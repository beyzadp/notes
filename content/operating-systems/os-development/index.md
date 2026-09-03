---
title: "Building a Bare-Metal RISC-V Kernel from Scratch"
---

notes from building a small RISC-V kernel from scratch in QEMU, loosely following [1000 line os](https://1000os.seiya.me/en/): the whole path from "empty linker script" to a kernel that boots, traps, schedules processes, talks to a virtio disk, and reads files off a custom filesystem. split into 9 parts, roughly in the order I actually built things, since each part leans on the one before it.

## the parts

1. **[[01-riscv-architecture|RISC-V Architecture]]**: QEMU setup, why RISC-V looks the way it does, the register set (GPRs/CSRs/FPRs), privilege levels (U/S/M-mode), a first look at Sv32 virtual memory, and the instruction formats. mostly reference material, not tied to this specific kernel.

2. **[[02-boot-sequence|Boot Sequence & Initialization]]**: the linker script, the `boot` function, `kernel_main`, and a full debugging writeup of a nasty bug where removing an infinite loop caused "Hello World" to print forever (turned out to be an uninitialized `ra` register).

3. **[[03-libc-and-io|Hardware Abstraction & Libc]]**: how output actually leaves the kernel: SBI calls through OpenSBI, building `printf` from scratch with no libc, and which bits of the standard library I had to reimplement myself.

4. **[[04-traps-and-interrupts|Trap & Interrupt Handling]]**: the `PANIC` macro (and why it's wrapped in `do { } while(0)`), what an exception actually is on RISC-V, and the full `kernel_entry` trap handler that saves/restores register state.

5. **[[05-memory-management|Memory Management]]**: the physical page allocator (a simple bump allocator), and virtual memory as a real security boundary: the `satp` register, the full Sv32 page table walk with worked examples, and page permission bits.

6. **[[06-process-management|Process Management]]**: the process control block, context switching in raw assembly, the cooperative scheduler (`yield()`), and why the trap handler needs `sscratch` once every process has its own kernel stack.

7. **[[07-syscalls|User Space & System Calls]]**: the actual trust boundary: what user mode can't do, how `ecall` bridges into the kernel, validating syscall arguments from untrusted user code, and the full syscall pipeline end to end.

8. **[[08-device-drivers|Device Drivers & Interrupts]]**: memory-mapped I/O, and the virtio-blk driver: device initialization, the status register handshake, virtqueues, and how a block read/write request actually flows through shared memory and a doorbell register.

9. **[[09-filesystem|File System Operations]]**: `MYFS`, a custom flat filesystem built on linked allocation: the read path, bounce buffers for safely crossing the user/kernel memory boundary, and the host-side tooling (`mkfs`/`extractfs`) needed to get files onto the disk image in the first place.
