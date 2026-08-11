---
title: "6. Process Management: The Illusion of Concurrency"
---

part of the [[index|os-development]] series.

The core trick is simple: one CPU can only run one instruction stream at a time, but if the OS swaps between programs fast enough, it **feels** like they are running together. this is the illusion of concurrency.

So process management is about time. the kernel saves the state of the current program, switches to another one, and restores its state. to the program, it looks like nothing happened. to the user, everything looks like it is running at once.

In a tiny kernel, this is not about fancy scheduling. it is about the basic ability to pause and resume execution safely and repeatedly.

## The Process Control Block and Kernel Stacks

The PCB is the OS's ledger for a process. it does not need to be huge. it just needs enough information to put the CPU back exactly where it left off: register values, stack pointer, program counter, and a few bookkeeping fields like state and PID.

this is my implementation:

```c
#define PROCS_MAX 8       // Maximum number of processes

#define PROC_UNUSED   0   // Unused process control structure
#define PROC_RUNNABLE 1   // Runnable process

struct process {
    int pid;             // Process ID
    int state;           // Process state: PROC_UNUSED or PROC_RUNNABLE
    vaddr_t sp;          // Stack pointer
    uint8_t stack[8192]; // Kernel stack
};
```

The important piece here is the **kernel stack**. every process needs its own private kernel stack because traps and context switches happen in kernel mode, and the kernel must have a safe place to save registers. if two processes shared one kernel stack, they would overwrite each other's saved state and the system would collapse.

So the rule is simple: one process, one kernel stack, one saved snapshot in the PCB. that is what makes "pause and resume" possible.

this is the implementation:

```c
struct process *create_process(uint32_t pc) {
    // Find an unused process control structure.
    struct process *proc = NULL;
    int i;
    for (i = 0; i < PROCS_MAX; i++) {
        if (procs[i].state == PROC_UNUSED) {
            proc = &procs[i];
            break;
        }
    }

    if (!proc)
        PANIC("no free process slots");

    // Stack callee-saved registers. These register values will be restored in
    // the first context switch in switch_context.
    uint32_t *sp = (uint32_t *)&proc->stack[sizeof(proc->stack)];
    *--sp = 0;            // s11
    *--sp = 0;            // s10
    *--sp = 0;            // s9
    *--sp = 0;            // s8
    *--sp = 0;            // s7
    *--sp = 0;            // s6
    *--sp = 0;            // s5
    *--sp = 0;            // s4
    *--sp = 0;            // s3
    *--sp = 0;            // s2
    *--sp = 0;            // s1
    *--sp = 0;            // s0
    *--sp = (uint32_t)pc; // ra

    // Initialize fields.
    proc->pid = i + 1;
    proc->state = PROC_RUNNABLE;
    proc->sp = (uint32_t)sp;
    return proc;
}
```


## Context Switching


A context switch is the moment the OS pauses one process and resumes another. it is not a high-level idea; it is a precise save/restore of CPU state.

The kernel does three mechanical steps:

1. **save** the current CPU registers onto the current process's kernel stack (or into its PCB).
2. **switch** the stack pointer to the next process's kernel stack.
3. **restore** the saved registers for the next process and return.

If every register is restored exactly, the new process continues as if it had never been interrupted. if even one register is wrong, you get instant corruption. 

this is my implementation:

```c
#define SAVE_CALLEE_REGS                                                       \
    "sw ra,  0  * 4(sp)\n"                                                     \
    "sw s0,  1  * 4(sp)\n"                                                     \
    "sw s1,  2  * 4(sp)\n"                                                     \
    "sw s2,  3  * 4(sp)\n"                                                     \
    "sw s3,  4  * 4(sp)\n"                                                     \
    "sw s4,  5  * 4(sp)\n"                                                     \
    "sw s5,  6  * 4(sp)\n"                                                     \
    "sw s6,  7  * 4(sp)\n"                                                     \
    "sw s7,  8  * 4(sp)\n"                                                     \
    "sw s8,  9  * 4(sp)\n"                                                     \
    "sw s9,  10 * 4(sp)\n"                                                     \
    "sw s10, 11 * 4(sp)\n"                                                     \
    "sw s11, 12 * 4(sp)\n"

#define RESTORE_CALLEE_REGS                                                    \
    "lw ra,  0  * 4(sp)\n"                                                     \
    "lw s0,  1  * 4(sp)\n"                                                     \
    "lw s1,  2  * 4(sp)\n"                                                     \
    "lw s2,  3  * 4(sp)\n"                                                     \
    "lw s3,  4  * 4(sp)\n"                                                     \
    "lw s4,  5  * 4(sp)\n"                                                     \
    "lw s5,  6  * 4(sp)\n"                                                     \
    "lw s6,  7  * 4(sp)\n"                                                     \
    "lw s7,  8  * 4(sp)\n"                                                     \
    "lw s8,  9  * 4(sp)\n"                                                     \
    "lw s9,  10 * 4(sp)\n"                                                     \
    "lw s10, 11 * 4(sp)\n"                                                     \
    "lw s11, 12 * 4(sp)\n"
```

```c
__attribute__((naked)) void switch_context(uint32_t *prev_sp,
                                           uint32_t *next_sp) {
    __asm__ __volatile__(
        // Save callee-saved registers onto the current process's stack.
        "addi sp, sp, -13 * 4\n" // Allocate stack space for 13 4-byte registers

        SAVE_CALLEE_REGS

        // Switch the stack pointer.
        "sw sp, (a0)\n" // *prev_sp = sp;
        "lw sp, (a1)\n" // Switch stack pointer (sp) here

        // Restore callee-saved registers from the next process's stack.
        RESTORE_CALLEE_REGS "addi sp, sp, 13 * 4\n" // We've popped 13 4-byte
                                                    // registers from the stack
        "ret\n");
}
```

Important lines:

- **`__attribute__((naked))`** tells the compiler not to add its own prologue/epilogue. this function is pure assembly and controls `sp` directly.

- **`addi sp, sp, -13 * 4`** reserves space to save callee-saved registers. we only save what the calling convention says must survive across calls.

- **`SAVE_CALLEE_REGS` / `RESTORE_CALLEE_REGS`** push and pop `ra` and `s0-s11`. these are the minimum registers that make a process resumable.

- **`sw sp, (a0)`** stores the current stack pointer into `*prev_sp`. this is how the PCB remembers where this process should resume.

- **`lw sp, (a1)`** loads the next process's stack pointer. from this moment on, we are operating on the next process's kernel stack.

- **`ret`** returns into the next process's saved `ra`, which is why the switch feels seamless.

## The Cooperative Scheduler

The scheduler is the policy layer that decides **who runs next**. in a tiny kernel, this can be as simple as "pick the next runnable process in a fixed list." it is not about fairness or priorities yet, just forward progress.

In cooperative scheduling, the kernel does not preempt. each process must **explicitly yield** the CPU (for example by calling `yield()` or by making a system call that returns to the scheduler). so the illusion of concurrency depends on processes being polite.

If no runnable process exists, the scheduler falls back to an **idle process**. the idle task does nothing except keep the CPU in a safe loop (often `wfi`) until a real process becomes runnable again.

So the cooperative scheduler is the simplest possible "traffic cop." it does not force a stop; it only switches when a process asks for it.

What we expect `yield()` to do:

1. **find the next runnable process.** scan the process table starting after `current_proc`. if none are runnable, fall back to `idle_proc`.
2. **decide if a switch is needed.** if `next == current_proc`, just return and keep running.
3. **save the current resume point.** treat the current `sp` as the resume address and store it in the current PCB.
4. **update ownership.** set `current_proc = next` so the kernel's view of who runs is correct.
5. **switch context.** call `switch_context(&prev->sp, &next->sp)` to save callee registers and restore the next process.

If those steps hold, `yield()` becomes the single, predictable doorway into scheduling.

this is the relevant part of the implementation:

```c
void yield(void) {
    /* omitted */

    __asm__ __volatile__(
        "csrw sscratch, %[sscratch]\n"
        :
        : [sscratch] "r" ((uint32_t) &next->stack[sizeof(next->stack)])
    );

    // Context switch
    struct process *prev = current_proc;
    current_proc = next;
    switch_context(&prev->sp, &next->sp);
}
```

- `csrw sscratch, ...` sets the **kernel stack pointer for the next process**. `sscratch` is the safe place the trap handler uses to swap stacks on entry, so we must update it before switching.
- `&next->stack[sizeof(next->stack)]` points to the **top of the next process's kernel stack** (stacks grow downward). that is the value we want in `sscratch`.
- `prev = current_proc` saves the old owner so we can write back its `sp` during the context switch.
- `current_proc = next` updates the global truth of who is running.
- `switch_context(&prev->sp, &next->sp)` performs the actual save/restore, using the two PCB stack pointers.

When to call `yield()`:

- after a process finishes a small unit of work and wants to be polite.
- inside long loops that would otherwise hog the CPU.
- after printing or doing simple I/O in this cooperative model, since no timer will preempt it.


here is a simple example of usage:

```c
void proc_a_entry(void) {
    printf("starting process A\n");
    while (1) {
        putchar('A');
        yield();
    }
}

void proc_b_entry(void) {
    printf("starting process B\n");
    while (1) {
        putchar('B');
        yield();
    }
}
```

we can call it from kernel_main

```c
void kernel_main(void) {
    memset(__bss, 0, (size_t) __bss_end - (size_t) __bss);

    printf("\n\n");

    WRITE_CSR(stvec, (uint32_t) kernel_entry);

    idle_proc = create_process((uint32_t) NULL);
    idle_proc->pid = 0; // idle
    current_proc = idle_proc;

    proc_a = create_process((uint32_t) proc_a_entry);
    proc_b = create_process((uint32_t) proc_b_entry);

    yield();
    PANIC("switched to idle process");
}
```

What this example is showing, line by line:

- `proc_a_entry` and `proc_b_entry` are tiny cooperative tasks. they do one small action (`putchar`) and then **yield** so the scheduler can pick someone else. without the `yield()`, a single loop would hog the CPU forever.

- `idle_proc = create_process((uint32_t) NULL);` creates a special process that represents "do nothing." it gives the scheduler a safe fallback so it never tries to run an invalid task.

- `idle_proc->pid = 0; current_proc = idle_proc;` makes the boot context pretend to be the idle process. this is important: when we first call `yield()`, the scheduler saves the **current stack pointer** into idle's PCB. later, when we switch back to idle, it looks like we simply returned from that same `yield()` call.

- `proc_a = create_process((uint32_t) proc_a_entry);` and `proc_b = create_process((uint32_t) proc_b_entry);` build initial stacks so the first context switch can `ret` directly into those entry functions.

- `yield();` is the first scheduling decision. since `current_proc` is idle, the scheduler picks a runnable process (A or B) and switches to it.

- `PANIC("switched to idle process");` is a guard. in a healthy run, we should not fall through here unless we switched back to idle and nothing else was runnable.

What was missing before: the trap handler assumed the current `sp` was safe to use. once we introduce per-process kernel stacks, that assumption breaks because a trap can arrive while `sp` still points to **user space**.

The correct approach is: **swap to the kernel stack first, then save registers.** we use `sscratch` to hold the kernel stack pointer for the running process and to temporarily park the user `sp`. this patches the `kernel_entry` function from [[04-traps-and-interrupts#Exception Handler]], this is why these lines are added and the old "save registers on current sp" path is removed:

```c
"csrrw sp, sscratch, sp\n"
"csrr a0, sscratch\n"
"sw a0,  4 * 30(sp)\n"
"addi a0, sp, 4 * 31\n"
"csrw sscratch, a0\n"
```

What this change buys us:

- `csrrw sp, sscratch, sp` swaps to the process's **kernel stack** and preserves the original user `sp` in `sscratch`.
- `csrr a0, sscratch` + `sw a0, 4 * 30(sp)` records the user `sp` in the trap frame so we can return correctly.
- `csrw sscratch, ...` resets `sscratch` to the top of the kernel stack for the next trap.

Limitation: saving registers uses the bottom 31 words of the kernel stack, so we do not support nested interrupts here.

## Context Isolation and Security (The sscratch Register)

This change brings us to this question:

Why do we reset the stack pointer?

The missing safety check was: **never trust `sp` on trap entry.** a trap can arrive from user mode with a corrupted or malicious stack pointer, and the handler will immediately start writing registers there. that can crash the kernel or trap in a loop.

Think about the three cases:

- **trap in kernel mode:** using the current `sp` is usually fine.
- **nested kernel trap:** we would overwrite the saved area, but we panic on nested traps, so it is acceptable in this tiny OS.
- **trap in user mode:** `sp` points to the user stack, which is untrusted. this is the dangerous case.

The attack is simple: a user program sets `sp` to an invalid address and triggers an exception. the trap handler tries to save registers to that bogus address, which itself triggers a fault, which re-enters the trap handler, and so on. this becomes an infinite trap loop and the kernel hangs.

That is why we **swap in the kernel stack from `sscratch` first**, then save registers. `sscratch` gives us a trusted stack pointer owned by the kernel, so the handler can always make forward progress.

Another design is to keep **two trap handlers**: one for kernel-mode traps that can reuse the current stack, and another for user-mode traps that always switch to a kernel stack. xv6 does this with `kernelvec` and `uservec`. in our minimal kernel, we keep one handler and rely on `sscratch` to make it safe.

The security habit is the same as everywhere else in the kernel: never trust user-controlled state, even something as basic as the stack pointer.
