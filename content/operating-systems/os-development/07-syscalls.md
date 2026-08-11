---
title: "7. User Space & System Calls: The Trust Boundary"
---

part of the [[index|os-development]] series.

This is where the kernel stops being the only program in the world and starts sharing the CPU with untrusted code. The core problem is simple: user programs should run, but they should never be able to damage the kernel or each other. To achieve this, we need a **trust boundary** backed by hardware.

The idea mirrors SBI calls, but one privilege level lower: SBI is Supervisor-mode (S-mode) asking Machine-mode (M-mode) for help. System calls (syscalls) are User-mode (U-mode) asking S-mode for help. The mechanism relies on the same trap door, but the threat model is entirely different because user code is assumed to be hostile.

## The Restricted Execution Environment

Dropping to user mode is the hardware saying: _"You can run, but you cannot control the machine."_

In U-mode, the CPU blocks privileged instructions, denies access to supervisor Control and Status Registers (CSRs), and refuses to touch pages that are not explicitly marked as user-accessible in the page tables. This is not a software convention; it is strictly enforced in silicon.

A user program **cannot**:

- Write to trap registers like `stvec` or `satp`.
- Disable or modify interrupts.
- Read or write kernel memory.
- Execute kernel-only pages.

If it tries to do any of these, the CPU immediately traps into the kernel. This restriction is the safety wall that lets us run unknown, untrusted code without handing over the keys to the system.

## The `ecall` Bridge & Syscall Validation

Because kernel addresses are not mapped as user pages, a user program cannot just call a kernel function. Even if it knew the exact memory address, the MMU would instantly fault on the instruction fetch. We need a controlled bridge.

`ecall` is that bridge. It is a deliberate trap instruction that tells the CPU to freeze user execution, cross the privilege boundary into S-mode, and jump to the kernel's pre-registered trap handler.

To use this bridge, the user program must pack a specific request (like reading a file or allocating memory) into the ABI registers:

- Syscall Number: The routing key identifying the requested service (usually in `a7`).
- Arguments: The payload, such as a buffer pointer and length (usually in `a0`-`a2`).
- Return Slot: The register where the kernel will drop the result (`a0`).


### Validating the Payload

Every syscall is an untrusted request. The most common attack vector is passing a fake pointer. for example, asking the kernel to "write this data" into a buffer that actually points to kernel memory.

Before acting on the request, the kernel *must* validate the payload:

* *Address Range:* The pointer and size must fall entirely within the user's address space.
* *Page Permissions:* The pages must be mapped and marked user-readable or user-writable, depending on the operation.
* *Alignment & Overflow:* The range calculation must not wrap around the address space.

If any check fails, the kernel rejects the syscall and returns an error code. Without validation, a single malicious syscall turns user code into kernel code.

---

## The Syscall Pipeline

Here is the exact sequence of events from the moment a user program requests a service to the moment it gets the result back.

1. *User code initiates request:* Runs in U-Mode.
The user program packs the syscall number and arguments into the standard ABI registers (e.g., `a0`-`a3`, `a7`) and executes the `ecall` instruction.


2. *Hardware traps to the Kernel:* CPU transitions to S-Mode.
The CPU stops user execution, saves the current Program Counter (PC) into `sepc`, sets `scause` to indicate an environment call, and jumps to the address stored in `stvec` (the kernel's trap entry point).


3. *Kernel saves state:*
The trap entry code saves all user registers into a trap frame on the kernel stack, ensuring the user's exact state can be restored later.


4. *Syscall dispatch and validation:* Hostile data assumption.
The kernel reads `scause` to confirm the trap was an `ecall`. It reads the syscall number and arguments from the saved trap frame, validates all pointers and ranges, and dispatches the request to the correct kernel service (e.g., `sys_write`).


5. *Execution and return preparation:*
The kernel service completes its work and writes the return value (or error code) into the `a0` slot of the saved trap frame. The kernel advances `sepc` by 4 bytes so the program doesn't infinitely re-execute the `ecall` instruction.


6. *Return to User Space:* Hardware drops back to U-Mode.
The kernel restores all registers from the trap frame and executes `sret`. Execution drops back to U-mode, and the user program resumes, reading its return value from `a0`.


> [!INFO]
> **The Stack Swap (`sscratch`):** Before saving the trap frame to memory, the kernel must use the `sscratch` register to swap the untrusted user stack pointer (`sp`) for a secure kernel stack pointer. Otherwise, a malicious user could set `sp` to a kernel address and overwrite critical data during the save.

> [!INFO]
> Here is an exact pipeline to add a new command, traveling from the user shell down to the physical hardware:
> 1. **The ID (`include/common.h`):** Define a new, unique system call number (e.g., `#define SYS_MYCMD 5`).
> 2. **The Wrapper (`src/user/user.c`):** Create a C function for the shell to call. This function uses the `syscall()` helper to fire the `ecall` assembly instruction, jumping to kernel mode.
> 3. **The Catcher (`src/kernel/kernel.c`):** Add your new ID to the `switch` statement inside `handle_syscall()`. This catches the jump and executes the actual privileged kernel logic.
> 4. **The Trigger (Shell `main`):** Add an `else if (strcmp(cmdline, "mycmd") == 0)` check inside your shell's infinite loop to parse the typed text and fire the wrapper function.
    
