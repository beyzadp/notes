---
title: "4. Trap & Interrupt Handling"
---

part of the [[index|os-development]] series.

_"During a trap, the standard calling convention is temporarily broken. The kernel must swap the user stack pointer for the kernel stack pointer using the `sscratch` register before it can safely save the context."_

## kernel panic

In bare-metal kernel work there is no higher layer to recover for us. When we hit an unrecoverable state (bad trap, corrupted state, invalid memory map), continuing execution just makes the system less debuggable and more dangerous. A panic gives us a single, deterministic escape hatch: print a clear failure message and stop the CPU so the failure is obvious and repeatable.

The panic path itself is a macro in `include/kernel.h`, so it can capture file and line information automatically and stay usable even before a full runtime exists:

```c
#define PANIC(fmt, ...) \
do { \
    printf("PANIC: %s:%d: " fmt "\n", __FILE__, __LINE__, ##__VA_ARGS__); \
    while (1) { \
    } \
} while (0)
```

Line by line:

- The `printf` line adds a `PANIC` prefix and the exact source location via `__FILE__` and `__LINE__`.
- The variadic `fmt` and `__VA_ARGS__` let me attach detailed context (CSR values, addresses, etc.).
- The `while (1)` loop halts the CPU so execution never returns to a corrupted state.

So after the macro mechanics, the real question is **when** to pull the lever. Panic is for **states that cannot be recovered safely**. In this codebase that includes:

- **Unexpected trap:** `handle_trap` fires with an unknown `scause` and we do not have a handler.
- **Memory exhaustion:** `alloc_pages` advances past `__free_ram_end`.
- **Corrupted invariants:** process state is invalid, or a stack pointer is outside the expected range.

The rule is simple: if continuing execution can silently corrupt state or make the bug non-deterministic, panic immediately.

Once it fires, `PANIC` prints **exactly once** and then halts in a tight loop. This is intentional:

- It prevents the kernel from returning into unknown control flow.
- It keeps the machine in a stable, inspectable state for a debugger.

If you want the CPU to sleep instead of burning cycles, you can replace the loop with `wfi`, but the core idea stays the same: **no return path**.

The message path is also worth stating explicitly, because it explains why panics sometimes appear "silent." The panic message relies on the same output path as everything else:

`printf` -> `putchar` -> `sbi_call` -> OpenSBI -> QEMU UART -> host terminal.

If this pipeline is broken (e.g., SBI not initialized or UART misconfigured), the panic still halts the CPU, but you will not see output. In that case, use GDB to inspect `scause`, `sepc`, and `stval` directly.

In practice, my panic sites are simple and early: `handle_trap` reports unexpected traps, and `alloc_pages` reports out-of-memory. A typical line looks like: 
`PANIC: src/kernel/kernel.c:358: unexpected trap scause=...`.

This is important because `PANIC` has minimal dependencies. It only needs `printf` and `putchar`, which makes it safe before the heap, scheduler, or virtual memory exist.

With that in mind, I also treat panic as a **structured breakpoint**:

- Print `scause`, `sepc`, and `stval` in `handle_trap` before panicking.
- Include addresses and sizes when allocator checks fail.
- Keep panic messages short so they remain reliable during early boot.

Why the `do { ... } while (0)` wrapper:

- A macro that expands to multiple statements can break `if/else` flow if it is not treated as a single statement.
- Wrapping the body in `do { ... } while (0)` makes the macro behave like one statement and consumes exactly one trailing semicolon.
- The condition is always false, so it runs exactly once, but it keeps scoping and control flow correct.

Examples tied to each point:

- **Broken `if/else` without a wrapper:**

```c
/*
 * Replace the two values.
 * The variable tmp is pre-defined.
 */
#define SWAP(x, y) \
  tmp = x; \
  x = y; \
  y = tmp

int x, y, z, tmp;
if (z == 0)
  SWAP(x, y);
```

Expands to:

```c
int x, y, z, tmp;
if (z == 0)
  tmp = x;
x = y;
y = tmp;
```

- **Broken else branch with a plain block:**

```c
/*
 * Replace the two values.
 * The variable tmp is pre-defined.
 */
#define SWAP(x, y) { tmp = (x); (x) = (y); (y) = tmp; }

if (x > y)
  SWAP(x, y); /* Branch 1 */
else  
  do_something(); /* Branch 2 */
```

Expands to:

```c
if (x > y)
  { tmp = (x); (x) = (y); (y) = tmp; }; /* the ; ends the if statement here */
else                                     /* error: 'else' without a previous 'if' */
  do_something();
```

- **Correct single-statement wrapper:**

```c
/*
 * Replace the two values.
 * The variable tmp is pre-defined.
 */
#define SWAP(x, y) \
  do { \
    tmp = (x); \
    (x) = (y); \
    (y) = tmp; } \
  while (0)
```

I use it from places like `handle_trap` and the allocator when I detect a fatal error.

## exception

An **exception** is a synchronous CPU event triggered directly by the instruction stream. 

In plain terms: 
- the CPU is executing an instruction, something about that exact instruction needs kernel attention (like a system call or an illegal access), so the CPU *pauses* the program at that point and forces a jump into the kernel. 
- The CPU checks `medeleg` to decide which mode should handle it. OpenSBI already set this so U/S exceptions go to the S-mode handler.
- It records context (`sepc` = where it stopped, `scause` = why it stopped, `stval` = extra detail like a bad address).
- The CPU loads the handler address from `stvec` and jumps there.
- The handler saves general-purpose registers, handles the exception, then restores them.
- When the kernel is done, it returns with `sret` and execution resumes from the saved `sepc` (or a corrected address).

| Register Name | Content |
| --- | --- |
| `scause` | Type of exception. The kernel reads this to identify the type of exception. |
| `stval` | Additional information about the exception (e.g., memory address that caused the exception). Depends on the type of exception. |
| `sepc` | Program counter at the point where the exception occurred. |
| `sstatus` | Operation mode (U-Mode/S-Mode) when the exception has occurred. |

### Exception Handler

```c
__attribute__((naked)) __attribute__((aligned(4))) void kernel_entry(void) {
    __asm__ __volatile__(
        "csrw sscratch, sp\n"
        "addi sp, sp, -4 * 31\n" // Allocate space for the trap_frame struct
        SAVE_ALL_REGS

        "csrr a0, sscratch\n"
        "sw a0, 4 * 30(sp)\n"

        "mv a0, sp\n"
        "call handle_trap\n"

        RESTORE_ALL_REGS "lw sp,  4 * 30(sp)\n"
        "sret\n");
}
```

Step-by-step:

1. The CPU jumps here because `stvec` points to `kernel_entry`.
2. `kernel_entry` is `naked`, so there is no compiler-generated prologue/epilogue.
3. It saves the current `sp` into `sscratch`.
4. It allocates space for a `trap_frame` by moving `sp` down.
5. It saves all registers into the trap frame (`SAVE_ALL_REGS`). This is the **context save**: caller-saved, callee-saved, and argument registers are all preserved so the interrupted code can resume exactly as it was.
6. It restores the original `sp` from `sscratch` into the trap frame slot.
7. It passes the trap frame pointer in `a0` and calls `handle_trap`.
8. When `handle_trap` returns, it restores all registers (`RESTORE_ALL_REGS`).
9. It restores the original `sp` and executes `sret` to resume execution. The restore is the mirror of the save: every register is put back so the CPU continues as if the trap never happened.

Why `sscratch`?

On trap entry, the CPU is still running with whatever `sp` was active in the interrupted context. I need a safe place to stash that pointer *before* I start pushing a trap frame, because the moment I move `sp` downward I am overwriting memory that belongs to the old stack. `sscratch` is the hardware-provided scratch register for exactly this handoff. I save the old `sp` there, build the trap frame on the kernel stack, and later restore the original `sp` so the interrupted code can resume. Without `sscratch`, I would have no reliable way to recover the previous stack pointer, and the return path could corrupt the caller's stack.

here is the handle_trap function:

```c
void handle_trap(struct trap_frame *f) {
    uint32_t scause = READ_CSR(scause);
    uint32_t stval = READ_CSR(stval);
    uint32_t user_pc = READ_CSR(sepc);

    PANIC("unexpected trap scause=%x, stval=%x, sepc=%x\n", scause, stval, user_pc);
}
```

Step-by-step:

1. The trap handler receives a pointer to the saved `trap_frame`.
2. It reads `scause` to identify the reason for the trap.
3. It reads `stval` to get the trap-specific value (often a faulting address).
4. It reads `sepc` to capture the program counter where the trap happened.
5. It panics with a formatted message that prints all three values.

Normally, `handle_trap` should branch on `scause` to handle syscalls, page faults, and interrupts. For now, I only read the CSRs and panic on any unexpected trap.

but how does `stvec` point kernel_entry?

`stvec` is the trap vector register. When traps are enabled, the CPU jumps to whatever address `stvec` holds. In this codebase, I set it to `kernel_entry` by writing the CSR from `kernel_main`. 

In `kernel_main` I added:

```c
WRITE_CSR(stvec, (uint32_t)kernel_entry); // new
__asm__ __volatile__("unimp"); // new
```

The first line wires the trap vector so any exception jumps into `kernel_entry`. The second line deliberately triggers an illegal instruction exception so I can verify the trap path is actually working end-to-end during bring-up.

To read and write CSRs cleanly, I use two small macros in `include/kernel.h`:

```c
#define READ_CSR(reg)                                                          \
    ({                                                                         \
        unsigned long __tmp;                                                   \
        __asm__ __volatile__("csrr %0, " #reg : "=r"(__tmp));                  \
        __tmp;                                                                 \
    })

#define WRITE_CSR(reg, value)                                                  \
    do {                                                                       \
        uint32_t __tmp = (value);                                              \
        __asm__ __volatile__("csrw " #reg ", %0" ::"r"(__tmp));                \
    } while (0)
```

`READ_CSR` returns the value of any CSR (like `scause`), and `WRITE_CSR` updates a CSR (like `stvec`) without repeating inline assembly at every call site.
