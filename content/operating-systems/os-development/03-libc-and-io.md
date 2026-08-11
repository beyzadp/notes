---
title: "3. Hardware Abstraction & Libc"
---

part of the [[index|os-development]] series.

## basic i/o and standard library functions

Before I can talk about libc-style I/O, I have to explain the firmware bridge that makes any output possible on bare metal: the SBI. 

My kernel runs in **Supervisor Mode**, so it cannot touch high-privilege hardware directly. The firmware layer, **OpenSBI**, runs in **Machine Mode**, and the SBI is the contract between them. It is the same idea as user-space syscalls into the Linux kernel, just one privilege level lower.

Execution starts in firmware (M-mode), then drops into my kernel (S-mode). When I need a privileged service like console output or timers, I issue an **`ecall`**. That instruction stops normal execution and traps into OpenSBI, which performs the operation and returns a result.

Unlike normal C calls, SBI arguments must be placed in specific registers. I bind the C variables to **`a0`-`a7`** using GCC register variables, then execute `ecall`. The return values come back in **`a0`** (error) and **`a1`** (value). This register contract is why the `sbi_call` wrapper exists.

Here is how I step through `sbi_call` in `src/kernel/kernel.c`:

1. **Bind arguments to registers.** Each C argument is assigned to a fixed ABI register using GCC register variables. That guarantees `arg0..arg5` live in `a0..a5`, and the function/extension IDs live in `a6`/`a7`.

2. **Issue the trap.** I execute `ecall`. This traps from Supervisor Mode to OpenSBI in Machine Mode.

3. **Read results.** After OpenSBI returns, `a0` contains the error code and `a1` contains the value. I wrap those in `struct sbiret` and return it to the caller.

```c
struct sbiret sbi_call(long arg0, long arg1, long arg2, long arg3, long arg4,
                       long arg5, long fid, long eid) {
    register long a0 __asm__("a0") = arg0;
    register long a1 __asm__("a1") = arg1;
    register long a2 __asm__("a2") = arg2;
    register long a3 __asm__("a3") = arg3;
    register long a4 __asm__("a4") = arg4;
    register long a5 __asm__("a5") = arg5;
    register long a6 __asm__("a6") = fid;
    register long a7 __asm__("a7") = eid;

    __asm__ __volatile__("ecall"
                         : "=r"(a0), "=r"(a1)
                         : "r"(a0), "r"(a1), "r"(a2), "r"(a3), "r"(a4), "r"(a5),
                           "r"(a6), "r"(a7)
                         : "memory");
    return (struct sbiret){.error = a0, .value = a1};
}
```

The inline assembly constraints are important: outputs mark `a0`/`a1` as clobbered by the firmware call, inputs tie all argument registers to the call site, and the `"memory"` clobber prevents reordering around the trap.


Once the register plumbing is in place, I still need a way to tell the firmware *what* I am asking for. The SBI uses a two-level routing scheme:

- **EID (Extension ID)** is the *category* of service I am calling (timer, reset, console, etc.). It is passed in **`a7`**.

- **FID (Function ID)** is the *specific action* inside that category (e.g., reboot vs. power off). It is passed in **`a6`**.

it's actually like syscalls.

For legacy SBI v0.1 console output, the mapping is even simpler: the **EID is `1`** for console putchar, and there is **no meaningful FID**. That is why my `putchar` path only needs to pass the character in `a0` and set `a7` to the legacy console EID.

Example: legacy console putchar (EID=1, no FID):

```c
void putchar(char c) {
    // a0 = character, a7 = legacy console EID (1)
    sbi_call(c, 0, 0, 0, 0, 0, 0, 1);
}
```

> Life of "Hello World" when I call console putchar:
> 
> 1. The kernel executes `ecall`, and the CPU jumps to the M-mode trap handler (`mtvec`) that OpenSBI set up during boot.
> 2. OpenSBI saves registers and transfers control to its C trap handler.
> 3. The handler dispatches the request by EID to the correct SBI implementation.
> 4. The 8250 UART driver in OpenSBI writes the character to the emulated device.
> 5. QEMU's 8250 UART emulation forwards the byte to standard output.
> 6. My terminal emulator renders the character on screen.
> 
> So Console Putchar is not magic; it is just a firmware driver path implemented in OpenSBI.


### details for creating `printf`

I build `printf` in `src/common/common.c` as a thin formatter over `putchar`. The core idea is to scan the format string one byte at a time and emit output immediately, so the function stays small and usable during early boot.

In `include/common.h` I also wire up the variadic machinery explicitly, since there is no libc to provide `<stdarg.h>`:

```c
#pragma once

#define va_list  __builtin_va_list
#define va_start __builtin_va_start
#define va_end   __builtin_va_end
#define va_arg   __builtin_va_arg

void printf(const char *fmt, ...);
```

These `__builtin_*` hooks are how I access the extra arguments that follow `fmt`. Without them, `printf` would not be able to pull values like `%d` or `%x` off the call stack/register save area in a freestanding build.


The structure I follow is:

1. **Fast path for plain characters:** If the current byte is not `%`, I send it directly to `putchar`.
2. **Specifier parsing:** When I hit `%`, I look at the next character to decide how to format the argument.
3. **Minimal specifiers:** I implement only what I need for debugging (`%s`, `%d`, `%x`). Everything else can be ignored or printed literally until I expand support.
4. **Integer formatting:** For `%d` and `%x`, I convert numbers to a temporary buffer in reverse, then emit the digits back to front with `putchar`.
5. **String formatting:** For `%s`, I iterate until `\0`, calling `putchar` on each character.

Because `putchar` ultimately calls `sbi_call`, every character goes through the SBI console path. This is slow but reliable.


### c standard library

The standard library is the baseline set of C utilities for strings, memory, formatted I/O, and variadic argument handling.

In normal hosted programs we rely on libc for these everyday tasks, because the OS provides a full runtime environment.

In a freestanding kernel that runtime does not exist yet, so we still need the same primitives but we must provide them ourselves. That is why I reimplement a tiny subset locally instead of linking the full libc.

Those familiar headers like `<stdio.h>`, `<string.h>`, and `<stdarg.h>` are just the API surface of libc. In user space they map to real implementations in the libc binary. At compile/link time this matters: if a program uses libc functions, the compiler emits calls to those symbols, and the linker must resolve them from a libc archive or shared library.

Quick step-by-step:

1. The compiler sees flags like `-ffreestanding` or `-nostdlib` and does not assume a hosted libc environment.
2. If I call `printf`/`memset`, it still emits external symbol references for them.
3. In a normal user-space build, the linker pulls those symbols from libc (`libc.a`/`libc.so`).
4. In my kernel build, there is no libc on the link line, so I must provide the symbols myself in `src/common/common.c`.

Why not just add a full libc: a full libc expects a hosted OS environment (syscalls, files, dynamic loader, memory management) that does not exist yet in early kernel bring-up, and it would either fail to link or crash at runtime. A tiny, known subset keeps the kernel bootable and predictable.

In the same spirit, I also define a few core constants and compiler builtins in `include/common.h`:

```c
#define true 1
#define false 0
#define NULL ((void *)0)
#define align_up(value, align) __builtin_align_up(value, align)
#define is_aligned(value, align) __builtin_is_aligned(value, align)
#define offsetof(type, member) __builtin_offsetof(type, member)
```

These exist because I cannot rely on `<stdbool.h>`, `<stddef.h>`, or other hosted headers. `true/false` and `NULL` give me minimal boolean and pointer conventions. The alignment helpers let me reason about page and buffer boundaries without writing ad-hoc bit math everywhere. `offsetof` is essential for low-level struct layout work, especially when building intrusive lists or interpreting memory-mapped data structures.

The concrete implementations I added in `src/common/common.c` are the small libc surface the kernel actually uses: `printf`, `putchar`, `memset`, `memcpy`, `strcmp`, and `strcpy`.
