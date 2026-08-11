---
title: "1. RISC-V Architecture"
---

part of the [[index|os-development]] series.

## qemu

**What is QEMU?**

It is a software emulator capable of simulating entire computer systems. 

**Why do we use it?**


- **Simplicity and Realism:** The `virt` machine provides a straightforward hardware layout that closely mirrors the behavior of actual physical devices.

- **Deep Debuggability:** When encountering low-level issues, QEMU provides absolute visibility. You can attach a debugger directly to the QEMU process to inspect the machine state, or read QEMU's source code to understand exactly how the virtual hardware is interpreting your instructions.


**How does it simply work?**

It works by programmatically replicating the components that make up a computer. QEMU handles the software emulation of discrete devices (CPU, memory, and peripherals), allowing operating systems and software to execute on top of it as if they were interacting with a physical machine.





## Core Architecture & Philosophy

RISC-V is an open, modular, load-store Instruction Set Architecture (ISA). Unlike x86, which is heavily CISC and monolithic, RISC-V separates its instruction set into a minimal base and optional standard extensions.

- **Base ISA:** Integer instructions (`RV32I`, `RV64I`, `RV128I`).
    
- **Standard Extensions:** Typically grouped as `G` (General Purpose), which includes `M` (Multiply/Divide), `A` (Atomics), `F` (Single-precision Float), and `D` (Double-precision Float). `C` (Compressed 16-bit instructions) is also standard for code density.

RISC-V is a **modular ISA** consisting of a mandatory **Base Integer Set** (e.g., RV64I) for fundamental execution and optional **Standard Extensions** (e.g., M, A, F, D, C) that add specific hardware circuits for specialized tasks.

Unlike monolithic architectures, this design allows hardware designers to minimize "silicon real estate" and power consumption by only implementing the physical logic gates required for the target application. For the systems programmer, this modularity dictates a strict contract: the compiler's target architecture (`-march`) must precisely match the extensions implemented in the hardware to avoid **Illegal Instruction traps**.

### openSBI

**OpenSBI** (Open-source Supervisor Binary Interface) is the standard open-source implementation of the RISC-V SBI specification. It acts as the **firmware** layer that sits between the hardware (running in Machine Mode) and your operating system kernel (running in Supervisor Mode).

**In short:** OpenSBI is the **BIOS/Firmware** of the RISC-V world. It handles the "dirty" hardware work so your kernel can stay "clean" and portable.


## register state


### registers


In RISC-V architecture, the registers are divided into three distinct sets that your kernel must manage.


#### 1. General-Purpose Registers (GPRs)

- **`x10-x17` (a0-a7):** Argument registers. Used to pass data to functions and SBI calls. `a0` and `a1` also hold return values.
    
- **`x8` (s0/fp):** Saved register or Frame Pointer.
    
- **`x5-x7`, `x28-x31` (t0-t6):** Temporary registers. These are "caller-saved," meaning a function can overwrite them without saving the previous value.



##### calling convention

**Caller-saved:** `ra` (x1), `t0-t6` (x5-x7, x28-x31), `a0-a7` (x10-x17).

**Callee-saved:** `sp` (x2), `s0-s11` (x8-x9, x18-x27), `gp` (x3), `tp` (x4).

When a function is called, the **Caller** (the code initiating the call) must assume that all `t` and `a` registers will be trashed. The **Callee** can start executing immediately. It doesn't have to waste cycles saving `t0` or `a1` to the stack if it needs to use them for a quick calculation.

Registers like `s0-s11` (Saved registers) are the opposite.

- If a function wants to use `s1`, it **must** save the original value to the stack first and restore it before returning (`ret`).


#### 2. Control and Status Registers (CSRs)


**Control and Status Registers (CSRs)** are a separate set of internal registers from the 31 general-purpose registers (`x1-x31`). While `x` registers are for math and data, CSRs are for **configuring the CPU's behavior** and **monitoring its state.**

- **Privilege Restricted:** Most CSRs can only be accessed by specific privilege levels. For example, a User-mode program cannot touch Supervisor-mode CSRs.
    
- **Special Instructions:** You cannot use `add` or `ld` on CSRs. You must use specific instructions:
    
    - `csrr`: Read a CSR into a general register.
    - `csrw`: Write a general register to a CSR.
    - `csrrw`: Swap values between a CSR and a general register.
    - `csrs csr, rs` (Set): Performs a bitwise OR. It sets specific bits in the CSR based on a mask in the GPR.
    - **`csrc csr, rs` (Clear):** Performs a bitwise AND-NOT. It clears specific bits.


##### important csr's

**`sstatus` (The Master Switch)**

This is a bitfield that tracks the CPU's current state.

- **SIE (Supervisor Interrupt Enable):** If this bit is 0, the CPU ignores all interrupts.
    
- **SPP (Supervisor Previous Privilege):** When a trap happens, the CPU records if you came from User or Supervisor mode here. When you run `sret`, the CPU looks at this bit to know which privilege level to return to.




**`stvec` (The Trap Vector)**

This CSR holds the address of your trap handler.

- **Direct Mode:** Every single trap/interrupt jumps to the exact same address.
    
- **Vectored Mode:** Different interrupts jump to different offsets in a table. Most simple kernels use **Direct Mode** and point this to their assembly `trap_entry`.



**`sepc` (The Return Pointer)**

When an exception (like a syscall) occurs, the CPU automatically copies the current `pc` (Program Counter) into `sepc`.

- **Crucial Detail:** For a **System Call**, `sepc` points to the `ecall` instruction itself. If you just run `sret`, you will execute `ecall` again in an infinite loop. the kernel must manually increment the value in `sepc` by 4 before returning.

#### 3. Floating-Point Registers (FPRs)

RISC-V has 32 floating-point registers (`f0-f31`), but they only exist if the hardware implements the `F` (single-precision) and/or `D` (double-precision) extensions, they're not part of the base integer ISA. they follow the same caller/callee-saved split as GPRs, just with an `f` prefix: `fa0-fa7` for arguments, `fs0-fs11` for saved values, `ft0-ft11` for temporaries.

we ignore them here because this kernel targets a build with no `F`/`D` extension enabled, so the compiler never emits floating-point instructions and there's no FPR state to save or restore. that's also why `switch_context` and the trap frame in this project only touch GPRs, if this kernel used floats, every context switch and trap would also need to save/restore the FPR file, which is extra work a bare-metal kernel usually doesn't need.

## Privilege Levels

RISC-V defines three primary **Privilege Levels** (also called "modes") to provide hardware-enforced security and isolation. These modes control what instructions can be executed and which memory regions or CSRs (Control and Status Registers) are accessible.


|Mode Name|Level|Description|
|---|---|---|
|**User (U-Mode)**|0|Lowest privilege. Restricted to application code; cannot access hardware or kernel memory directly.|
|**Supervisor (S-Mode)**|1|Intermediate privilege. Where the OS kernel runs. Manages virtual memory, processes, and handles interrupts.|
|**Machine (M-Mode)**|3|Highest privilege. Full hardware access. Runs firmware (OpenSBI) and handles low-level hardware initialization.|

even if this is not looking that important, while implementing functions this can be a big headache.

The CPU moves between levels through a mechanism called **trapping**.

- **Vertical Movement:** You cannot simply "jump" to a higher privilege. You must execute an `ecall` (environment call), which triggers a trap. The hardware then forces the CPU into a higher mode and jumps to a pre-defined address (the trap handler).
    
- **The Return:** To go back down (e.g., from Kernel to User), you use specialized instructions like `sret` (Supervisor Return) or `mret` (Machine Return). These instructions atomically restore the previous privilege level and resume execution.

here is an example:

> Lets say I'm a user and I want to print data. Since I don't have the privilege to access the hardware, I execute an `ecall` to trap into the kernel. The kernel receives this, realizes it needs to talk to the hardware, and makes an SBI call (another `ecall`) to OpenSBI. OpenSBI receives the call and executes it in Machine Mode.


- **User `ecall`** = A **System Call** (U $\rightarrow$ S).
- **Kernel `ecall`** = An **SBI Call** (S $\rightarrow$ M).



## Memory Management Unit (Sv32 Theory)

Sv32 is the 32-bit page-table scheme: two levels of page tables, 4KB pages, and a 32-bit virtual address split into indexes plus an offset. it is a compact way to understand how the MMU "walks" page tables, translates addresses, and enforces permissions before we deal with deeper schemes like Sv39. the detailed step-by-step walk and permission checks live in [[05-memory-management#Address Translation & Permissions]].




## instructions


### instruction format

In a base RISC-V set (like `RV64I`), every instruction is exactly **32 bits** long and must be aligned to a 4-byte boundary (unless using the `C` extension).

The bits are divided into fields:

- **Opcode:** Tells the CPU what _type_ of operation to do (e.g., Load, Store, Branch).
    
- **rd (Destination Register):** Where the result is stored.
    
- **rs1 / rs2 (Source Registers):** The input registers for the operation.
    
- **Immediate:** A constant value "baked" directly into the instruction (e.g., the `4` in `addi a0, a0, 4`).

### Instruction Categories (Functional)

**A. Arithmetic & Logic**

These happen entirely inside the CPU.

- `add`, `sub`, `xor`, `or`, `and`
    
- `sll`, `srl` (Shifts)
    

**B. Load & Store (The Only Way to Touch RAM)**

RISC-V is a **Load-Store Architecture**. You cannot add a number directly to a memory address. You must:

1. **Load** the value from RAM into a register (`ld`).

2. **Add** the number in the register (`addi`).

3. **Store** the result back to RAM (`sd`).


**C. Control Flow**

- **Branches:** `beq` (branch if equal), `bne` (not equal), `blt` (less than).

- **Jumps:** `jal` (Jump and Link - used for calling functions) and `jalr` (Jump and Link Register - used for returning from functions via the `ra` register).


**D. System Instructions**

These are critical for your bare-metal OS:

- **`ecall`:** The "doorbell" to trigger a trap to a higher privilege level.
    
- **`ebreak`:** Used by debuggers to stop execution.
    
- **`mret` / `sret`:** Returns from a trap to a lower privilege level.
    
- **`csrr` / `csrw`:** Reading/Writing the CSRs we discussed earlier.
    

#### Privileged instructions

| **Instruction** | **Description**                                                                        |
| --------------- | -------------------------------------------------------------------------------------------------------- |
| `csrr`          | **Read from CSR:** Reads a value from a Control and Status Register into a general register.             |
| `csrw`          | **Write to CSR:** Writes a value from a general register into a Control and Status Register.             |
| `csrrw`         | **Read/Write CSR:** Reads from and writes to a Control and Status Register at the same time.             |
| `sret`          | **Return from trap handler:** Restores the program counter, operation mode, etc., after handling a trap. |
| `sfence.vma`    | **Clear TLB:** Clears the Translation Lookaside Buffer.                                                  |

## debugging

typical workflow for debugging this kernel: run qemu with `-s -S`, which starts it paused and opens a gdb server on `tcp::1234` (`-s` is shorthand for `-gdb tcp::1234`, `-S` means "don't start executing until told to").

then from another terminal:

```
gdb-multiarch build/kernel.elf
(gdb) target remote :1234
(gdb) break kernel_main
(gdb) continue
```

useful commands from there:

- `info registers` to see all the GPRs at once.
- `p $sepc`, `p $sstatus`, etc, qemu's gdbstub exposes RISC-V CSRs as regular gdb registers, so you read them the same way as GPRs.
- `x/10xw <addr>` to dump raw memory as hex words, useful for checking page table entries or a trap frame by hand.
- `stepi`/`nexti` to step one instruction at a time, needed for `naked` functions since they have no line info for `step`/`next` to use.
- `disassemble` to see the actual instructions around the current pc, handy when `sepc` points somewhere you don't recognize.

## Memory Management Unit (Sv39 Theory)

Sv39 is the equivalent scheme for RV64: three levels of page tables instead of two, still 4KB pages, but a 39-bit virtual address instead of 32-bit (the top 25 bits of the 64-bit VA register have to be sign-extended copies of bit 38, or the CPU rejects the address).

the split is `VPN[2]:VPN[1]:VPN[0]:offset` = 9:9:9:12 bits, instead of Sv32's `VPN[1]:VPN[0]:offset` (10:10:12). the walk is the same idea as Sv32, just one extra hop: read `satp` for the root table, index with `VPN[2]` to get the level-1 table, index that with `VPN[1]` to get the level-0 table, index that with `VPN[0]` to get the final PTE, then add the page offset. the permission bits (`V`/`R`/`W`/`X`/`U`/`G`/`A`/`D`) mean the same thing as Sv32, the PTE format is just wider (64-bit entries instead of 32-bit).
