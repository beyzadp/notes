---
title: "Virtual Memory and Copy-on-Write fork()"
---

When your C program starts, the OS gives it a completely isolated **Virtual Address Space**. this exists mostly for isolation, if processes touched physical RAM directly, one bad pointer in any program could read or overwrite another process's memory, or the kernel's. giving every process its own virtual space, translated through the MMU, means it can only ever touch what the kernel explicitly mapped in for it. it's also what makes `fork()`'s CoW trick possible in the first place, since two separate virtual spaces can be pointed at the same physical memory without either process knowing.

This space is divided into segments:

- **Text:** Your compiled machine code. read + execute, not writable.
- **Data / BSS:** Global and static variables (this is where an unnamed `sem_t` lives if you declare it globally). read + write.
- **Heap:** Dynamically allocated memory (`malloc`). read + write, grows upward.
- **Stack:** Local variables and function call frames. read + write, grows downward.

(there's also the memory-mapped region, shared libraries and `mmap()`'d files, sitting between the heap and the stack, not covered here yet.)

The CPU contains a hardware component called the **Memory Management Unit (MMU)**. The MMU uses a **Page Table** to translate your program's Virtual Addresses (VA) into actual Physical Addresses (PA) on the RAM chips, in fixed-size chunks called pages (usually 4KB). Your program _only_ ever sees Virtual Addresses.

Walking the page table on every single memory access would be slow, so the CPU caches recent VA->PA translations in a **TLB** (Translation Lookaside Buffer). this matters for CoW later: whenever the OS changes what a Virtual Address points to, it also has to invalidate the stale TLB entry, or the CPU might keep using the old translation.

When you call `fork()`, the kernel creates a new process. It simply duplicates the parent's **Page Table**. At the exact microsecond `fork()` returns, both the Parent and the Child have the exact same Virtual Addresses, and their Page Tables point to the exact same Physical Addresses.

Why does the OS do this? The short answer is: **Extreme laziness for the sake of speed.** Here is why immediately giving the child its own physical memory would be a disaster for system performance. 

**1. The Cost of Copying Everything**

Imagine you are running a database server that is currently using 4 Gigabytes of RAM.

If that database calls `fork()` to handle a new client, and the OS immediately creates separate physical addresses for everything, the CPU has to physically halt and copy 4 Gigabytes of data from one set of RAM chips to another.

Copying gigabytes of memory takes a massive amount of CPU cycles. Your entire system would freeze up every time a new process was created.


**2. The `exec()` Reality (The Waste)**

The OS designers realized something critical about how UNIX works: 99% of the time, immediately after a program calls `fork()`, the child process turns around and calls `exec()`.

`exec()` is a system call that says, "Destroy my current memory entirely and load a brand new program (like `/bin/ls` or a python script) into my space."

If the OS had just spent a massive amount of time and energy perfectly copying 4 Gigabytes of physical RAM for the child, and a microsecond later the child calls `exec()` and throws all that copied memory in the garbage, it would be a tragic waste of system resources.


**3. The "Lazy" Solution: Copy-on-Write**

Because of this, the OS takes a gamble.

When you call `fork()`, the OS says: "I'm not going to copy the physical RAM. That takes too long. I'm just going to give the child a duplicate map pointing to the parent's RAM."

- **If the child calls `exec()`:** The gamble paid off! The OS just deletes the child's map, creates a new one for the new program, and didn't waste any time copying the parent's memory.
    
- **If the child just wants to READ data:** The gamble also pays off. Both parent and child can read the same physical memory without any issues.
    
- **If the child tries to WRITE data:** The OS finally steps in. It says, "Okay, you actually want to change this specific 4-Kilobyte chunk of memory. _Now_ I will copy just this tiny piece into a new physical address for you."

a write to a read-only page is what triggers this in the first place: the CPU raises a **page fault**, a general exception for "this memory access isn't allowed as-is," which the kernel has to inspect and decide what to do with. not every page fault is a CoW situation, the same mechanism also covers demand-paging (page not present yet) and genuinely illegal accesses, which is what turns into a `SIGSEGV`. so the first thing the handler has to do is figure out *which* of those this is.

Then the Kernel Resolves the Fault (The Actual CoW):

This is where the OS checks its own internal, high-level records (which are separate from the CPU's hardware Page Table, this bookkeeping is usually called something like a VMA, virtual memory area, and it's what lets the kernel tell "this is a legit CoW page" apart from "this write is actually illegal").

1. The OS checks its records for the child process and says, "Wait, Virtual Address `0x55eb43a0` is _supposed_ to be writable memory. It's only marked Read-Only right now because of a `fork()`."

2. **Refcount check:** the OS also checks how many page tables still point at Frame #100. if this process is the only one left (say, the sibling that shared it already exited or wrote its own copy), there's no one left to protect the frame from, so the OS can just flip the permission bit back to Read/Write in place and skip the copy entirely.

3. **Allocation:** if the frame is still shared, the OS goes to the computer's free physical RAM and claims a brand new, empty 4-Kilobyte block (let's call it Frame #200).

4. **The Copy:** The OS copies all 4,096 bytes from the old Frame #100 into the new Frame #200.

5. **Updating the Map:** The OS changes the child's Page Table. It updates Virtual Address `0x55eb43a0` so it now points to Frame #200 instead of Frame #100.

6. **Fixing Permissions:** The OS changes the permission on that specific Page Table entry from "Read-Only" to "Read/Write".

7. **Flushing the TLB:** the CPU might still have the old VA->Frame #100 translation cached, so the OS has to invalidate that entry before the child's next access, otherwise it could still hit the old, read-only frame.

## todo

- N-way sharing: what happens if a child forks again before writing (chained refcounting).
- CoW outside of `fork()`, e.g. `mmap(MAP_PRIVATE)` vs `MAP_SHARED`.
- a practical section: inspecting real CoW pages via `/proc/[pid]/maps` or similar.
- `vfork()`, the older mechanism CoW replaced.

