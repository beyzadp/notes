---
title: "5. Memory Management: Bookkeeping and Isolation"
---

part of the [[index|os-development]] series.

At boot, RAM is just a big flat array of bytes. there is no "kernel memory" or "user memory." the CPU can read or write any address, and it will happily do it. so the OS has to invent structure and enforce it.

In a tiny kernel, memory management is basically two jobs:

- **bookkeeping:** track which physical addresses are free and which are already reserved for the kernel, stacks, page tables, or programs.
- **isolation:** make sure user code cannot touch kernel data, even if it tries.

This is one of the OS "illusions":

- **illusion of private memory:** each process feels like it owns a clean, continuous space.
- **illusion of safety:** the kernel knows user code cannot reach into protected regions.

We are not building a fancy allocator or swapping system here. we just need the minimum ledger to avoid overlap, and the hardware MMU to enforce the trust boundary.

## Physical Page Allocation



We need a bare-minimum tracking system because the CPU will not stop us from handing out the same memory twice. if the kernel gives two subsystems the same physical bytes, you get silent corruption and random crashes. so even in a tiny kernel, we need a ledger.

The simplest ledger is page-based. we divide RAM into fixed 4KB chunks (pages) and track which pages are **free** and which are **in use**. this avoids the complexity of a full heap allocator with variable-size blocks and fragmentation rules.

Why 4KB?

- the MMU already thinks in 4KB pages, so our allocator should match the hardware's granularity.
- page tables, permissions, and address translation all operate on page boundaries.

So the allocator's job is not to be clever. it just walks a simple list or bitmap of pages:

- **free page:** can be handed out to the kernel or mapped to a user process.
- **used page:** already assigned, do not touch.

This is enough for a 1000-line kernel. we are not solving fragmentation or fast lookup. we are just making sure every page has exactly one owner.

> Modern OSes do the same bookkeeping but at scale: a buddy allocator for pages plus slab/SLUB caches for small objects to reduce fragmentation and speed up allocations.

When we start doing page allocation, we need the linker to mark a clean "free RAM window" for us. that is why we add this chunk to the linker script:

```c
. = ALIGN(4096);
__free_ram = .;
. += 64 * 1024 * 1024; /* 64MB */
__free_ram_end = .;
```

This is not a real section like `.text` or `.data`. it is just a reservation. we move the location counter forward and create two symbols that describe a safe range the allocator can manage.

- **`. = ALIGN(4096);`** page allocators and page tables work in 4KB units. aligning here makes the first page clean and prevents half-pages at the start.

- **`__free_ram = .;`** this is the first usable byte after the kernel image and the stack. the allocator treats this as the start of the free list.

- **`. += 64 * 1024 * 1024;`** this fences off a fixed-size pool (64MB here). it is not allocating or zeroing memory; it is just saying "this range is ours to manage." the size is a policy choice for a tiny kernel.

- **`__free_ram_end = .;`** this is the hard stop. the allocator never hands out pages past this address.

Why in the linker? because the linker is the only place that knows the exact end of the kernel image. it can safely define "free memory starts here" without guessing. the kernel then reads `__free_ram` and `__free_ram_end` to build its first simple memory ledger.

After that, the simplest allocator is just a "bump pointer." keep one static pointer to the next free page, hand out `n` pages by moving it forward, and panic if it crosses the end. it always returns 4KB-aligned addresses (because the linker aligned the start), and it zeroes pages so uninitialized memory does not leak garbage.

The limitation is obvious: this allocator cannot free. but for a 1000-line kernel, that tradeoff is fine. it gives us predictable, deterministic allocation without the complexity of a real heap.

So far in memory management, we only carved out a free RAM window in the linker and built a bump-pointer page allocator on top of it. that is enough to allocate clean, page-aligned memory during early boot, but it does not free or reuse anything yet.

## Virtual Memory as a Security Boundary

We do not allow user programs to touch physical addresses directly because that would let them read or overwrite **any** memory in the system, including the kernel itself. one bad pointer could corrupt kernel state, and one malicious program could steal data from another.

Virtual memory fixes this by inserting a hardware-enforced sandbox. each process gets its own virtual address space, and the MMU only translates addresses that the kernel has explicitly mapped for that process. anything outside that map simply does not exist from the process's point of view.

So the "boundary" is not a convention, it is a hardware rule: user code cannot see kernel pages, cannot write to read-only pages, and cannot execute non-executable pages. if it tries, the CPU raises a page fault and traps into the kernel.

### Address Translation & Permissions

This is where the MMU turns a virtual address into a physical address and enforces access rules for each page.

**satp register:**

`satp` is the switch that turns virtual memory on and points the MMU at the root page table. the `MODE` bit tells the CPU whether to translate addresses at all, the `PPN` tells it where the root page table lives in physical memory, and `ASID` tags the current address space so different processes do not clash in the TLB.

| Bit Range | Size | Name | Purpose                                    |
| ------------- | -------- | -------- | ---------------------------------------------- |
| 31        | 1 bit    | `MODE`   | Enables or disables virtual memory.            |
| 30 ... 22 | 9 bits   | `ASID`   | Address Space Identifier (process ID tagging). |
| 21 ... 0  | 22 bits  | `PPN`    | Physical Page Number of the Root Page Table.   |

#### Sv32 Page Table Walk

This is the MMU's step-by-step lookup: it follows pointers through page tables using pieces of the virtual address until it finds the final physical page.

**1. Slice the Virtual Address**

Before doing anything, the hardware splits the 32-bit Virtual Address into three pieces:

- `VPN[1]` (Top 10 bits): The index for the Level 1 table.
- `VPN[0]` (Middle 10 bits): The index for the Level 0 table.
- `Page Offset` (Bottom 12 bits): The exact byte location within the final 4KB page.

> For Virtual Address **`0x40001234`**:
> 
> - `VPN[1]` = **256**
> - `VPN[0]` = **1**
> - `Page Offset` = **`0x234`** (564 in decimal)

**2. Find the Root Table (`satp`)**

- Read the `satp` register to get the Physical Page Number (PPN) of the root table.
- Shift that PPN left by 12 (`<< 12`) to find the **Level 1 Base Address**. (Because Sv32 memory pages are exactly 4 KiB, their true physical addresses always end in 12 zeros. To save space in the 32-bit `satp` register, the hardware drops those zeros and stores only the remaining top bits as a Physical Page Number (PPN). When the CPU actually needs to read memory, it shifts the PPN left by 12 (`<< 12`) to append those zeros back, perfectly restoring the true physical byte address.)


> **Example Application:**
> 
> The `satp` register holds **`0x80401000`**.
> - Extract the PPN: **`0x001000`**
> - Base Address = `0x001000 << 12` = **`0x01000000`**


**3. Jump to the Level 1 Entry**


- Level 1 Base Address + `(VPN[1] * 4)`. (Multiply by 4 because each entry is 4 bytes).
- Go to that exact physical address and read the **Level 1 Page Table Entry (PTE)**.


> **Example Application:**
> - L1 PTE Address = `0x01000000 + (256 * 4)` = **`0x01000400`**
> - The CPU reads memory at `0x01000400` and finds the value **`0x00800001`**.


**4. Find the Level 0 Table**

- Strip the flags off the Level 1 PTE to get its PPN.
- Shift that PPN left by 12 (`<< 12`) to find the **Level 0 Base Address**.

> **Example Application:**
> 
> - Strip flags from L1 PTE (`0x00800001 >> 10`) to get PPN: **`0x002000`**
> - Level 0 Base Address = `0x002000 << 12` = **`0x02000000`**

**5. Jump to the Level 0 Entry**

- Take the Level 0 Base Address and add `(VPN[0] * 4)`.
- Go to that exact physical address and read the **Level 0 Page Table Entry (PTE)**.
    
> **Example Application:**
> 
> - L0 PTE Address = `0x02000000 + (1 * 4)` = **`0x02000004`**
> - The CPU reads memory at `0x02000004` and finds the value **`0x00C000DF`**.


**6. Form the Final Physical Address**

- Strip the flags off the Level 0 PTE to get the final target PPN.
- Shift that target PPN left by 12 (`<< 12`) to get the base address of the actual memory page.
- Add your original 12-bit `Page Offset` to hit the exact byte in physical RAM.

> **Example Application:**
> 
> - Strip flags from L0 PTE (`0x00C000DF >> 10`) to get Target PPN: **`0x003000`**
> - Target Page Base = `0x003000 << 12` = **`0x03000000`**
> - Final Physical Address = `0x03000000 + 0x234` = **`0x03000234`**

Quick summary: split VA -> walk page tables from `satp` -> reach PTE -> apply permissions -> combine PPN + offset for the final PA.



#### permissions

Permissions are just bits in the final PTE. if a page is not marked readable, writable, or executable, the CPU will raise a fault on that access. the `U` (user) bit is the key boundary: user code can only touch pages that are explicitly marked user-accessible, while kernel-only pages stay invisible to user mode.

Here is the complete 32-bit breakdown of an Sv32 Page Table Entry (PTE):

| Bit Range | Size | Name       | Purpose                                                                    |
| ------------- | -------- | -------------- | ------------------------------------------------------------------------------ |
| **31 ... 10** | 22 bits  | `PPN`          | Physical Page Number (Points to the next page table or the final memory leaf). |
| **9 ... 8**   | 2 bits   | `RSW`          | Reserved for Software (Ignored by hardware; free for OS custom tracking).      |
| **7**         | 1 bit    | `D` (Dirty)    | Set by hardware the first time the page is written to.                         |
| **6**         | 1 bit    | `A` (Accessed) | Set by hardware the first time the page is read, written, or executed.         |
| **5**         | 1 bit    | `G` (Global)   | Keeps the mapping active across all process spaces (ignores ASID changes).     |
| **4**         | 1 bit    | `U` (User)     | If 1, user-mode applications can access. If 0, supervisor-mode only.           |
| **3**         | 1 bit    | `X` (Execute)  | Grants permission to fetch and execute CPU instructions from this page.        |
| **2**         | 1 bit    | `W` (Write)    | Grants permission to modify data on this page.                                 |
| **1**         | 1 bit    | `R` (Read)     | Grants permission to read data from this page.                                 |
| **0**         | 1 bit    | `V` (Valid)    | Master switch. If 0, the entry is ignored and throws a Page Fault exception.   |

we can define macros:

```
#define SATP_SV32 (1u << 31)
#define PAGE_V    (1 << 0)   // "Valid" bit (entry is enabled)
#define PAGE_R    (1 << 1)   // Readable
#define PAGE_W    (1 << 2)   // Writable
#define PAGE_X    (1 << 3)   // Executable
#define PAGE_U    (1 << 4)   // User (accessible in user mode)
```

#### implementation

**map_page function**


what do we expect from this function:

- it should take the first-level page table (`table1`), the virtual address (`vaddr`), the physical address (`paddr`), and page table entry flags (`flags`)
- we need to check if vaddr and paddr are valid. a valid page address **must** have its bottom 12 bits set to zero.
- for extracting `VPN[1]` we need to extract the top 10 bits using right shift and mask it to be sure.
- if not exist, we need to create the table.
- then we can extract the `VPN[0]` by shifting right by 12. with the mask `& 0x3ff` we can be sure the only thing left is `VPN[0]` bits.
- we jump to the level 0 table. 
- we need to reread the level1 PTE to have the PPN. we do this by shifting `table1[VPN[1]]` by 10 bits. then we convert the address.
- after we hit our final destination, we shift it left by 10 bits to build the PTE data structure, we apply permissions and turn the valid bit to 1. 
- now the virtual address is mapped.

which can be written as:

```c
void map_page(uint32_t *table1, uint32_t vaddr, paddr_t paddr, uint32_t flags) {
    if (!is_aligned(vaddr, PAGE_SIZE))
        PANIC("unaligned vaddr %x", vaddr);

    if (!is_aligned(paddr, PAGE_SIZE))
        PANIC("unaligned paddr %x", paddr);

    uint32_t vpn1 = (vaddr >> 22) & 0x3ff;
    if ((table1[vpn1] & PAGE_V) == 0) {
        // Create the 1st level page table if it doesn't exist.
        uint32_t pt_paddr = alloc_pages(1);
        table1[vpn1] = ((pt_paddr / PAGE_SIZE) << 10) | PAGE_V;
    }

    // Set the 2nd level page table entry to map the physical page.
    uint32_t vpn0 = (vaddr >> 12) & 0x3ff;
    uint32_t *table0 = (uint32_t *) ((table1[vpn1] >> 10) * PAGE_SIZE);
    table0[vpn0] = ((paddr / PAGE_SIZE) << 10) | flags | PAGE_V;
}
```

we also need to change the PCB and add a symbol to the linker script named `kernel_base`. this is because we want the os to only map the actual RAM where the kernel and free memory live, do not map the hardware devices into the process.

we can map the kernel by these lines:

```c
    // Map kernel pages.
    uint32_t *page_table = (uint32_t *) alloc_pages(1);
    for (paddr_t paddr = (paddr_t) __kernel_base;
         paddr < (paddr_t) __free_ram_end; paddr += PAGE_SIZE)
        map_page(page_table, paddr, paddr, PAGE_R | PAGE_W | PAGE_X);
```

we also need to change the yield function to make the context switching include process's page table.
