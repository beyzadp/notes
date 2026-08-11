---
title: "Linux Signals"
---

signals are how the kernel (or another process) can asynchronously interrupt a process to tell it something happened: a crash, a Ctrl+C, a child exiting, a timer firing, or just another process explicitly poking it. without signals a process would have to constantly poll for these things everywhere in its code; signals let the kernel do that "did something happen" check and just jump into your code when it matters.

this note covers how that delivery actually works under the hood: the difference between standard and real-time signals, how dispositions work, and then the actual `sigaction`-based programming side of catching and handling them properly.

## the basics

### Standard Signals

standard signals (1-31) are the traditional unix ones: faults, terminal interactions, lifecycle events, each with a default action like terminating or dumping core.

they don't queue. if `SIGINT` (2) is sent, bit 2 in the pending mask just flips to 1. if another `SIGINT` arrives before the first is handled, that bit's already 1, so the second one is effectively lost. the kernel only knows _at least one_ happened.

> for example: Receipt of `SIGCHLD` translates simply to: _"At least one child process has changed state; inspect the process table."_ That's why we need a harvesting loop.

### Real-time Signals

real-time signals are for application-level logic rather than kernel faults, and unlike standard signals they queue: multiple instances stack up and get delivered in order, each with its own payload.

> If you send three `SIGRTMIN` signals to a process, the process will execute its handler exactly three times. They are delivered in the order they were received.

they can carry data too. `sigqueue()` lets you attach a `sigval` (an int or a `void *`) when sending one, and the receiving process can read it back out of the `siginfo_t` struct when it catches the signal, basically a lightweight form of IPC.

### Signal Dispositions

a disposition is just what a process does when a signal arrives, every process keeps a table of function pointers/flags for this. three options:

- `SIG_DFL` (default): kernel does whatever the default action is. `SIGSEGV` dumps core and terminates, `SIGCHLD` gets ignored by default.
- `SIG_IGN` (ignore): you tell the kernel to just drop it, it never interrupts the process.
- custom handler: you register a function pointer (usually via `sigaction`). the kernel pauses the process's main thread, sets up a stack frame, and forces the instruction pointer (`%rip`) to jump into your C function.

`SIGKILL` (9) and `SIGSTOP` (19) are the exceptions: you can't catch, block, or ignore either one, for system stability. when `SIGKILL` hits, the kernel just rips the process out of memory immediately, no chance for user-space to run any cleanup code.

## how delivery actually works

when a hardware exception happens, or user-space calls something like `kill(pid, sig)`, the kernel locks the thread group's shared `sighand_struct` and updates the `pending` bitmask (plus the pending list too, if it's a queued real-time signal). the handler doesn't run yet though, the kernel's just recording that the event happened.

it only actually checks the `task_struct` for pending, unblocked signals at one specific moment: when a thread is transitioning from kernel-space back to user-space. that happens

1. right after returning from a standard system call, or
2. right after returning from a hardware interrupt (like a scheduler context switch).

if pending-vs-blocked comes out non-zero at that point, delivery happens immediately, overriding the normal return path.

running your handler code is trickier since the kernel can't safely execute user-supplied code in Ring 0, so instead it manipulates the user stack:

1. it builds a `sigframe` on the user stack, saving the thread's exact CPU registers (`%rip`, `%rsp`, etc) into a `ucontext_t` struct.
2. it overwrites the saved instruction pointer to point at your handler function.
3. the return address gets set to the signal trampoline (`__vdso_rt_sigreturn`), a snippet of code injected via the vDSO.
4. once your handler finishes and `ret`urns, it lands in the trampoline, which fires the `rt_sigreturn` syscall. the kernel reads the `ucontext_t` back, restores the CPU state exactly, and resumes wherever it left off.

## programming

```c
struct sigaction {
    union {
        void     (*sa_handler)(int);
        void     (*sa_sigaction)(int, siginfo_t *, void *);
    } __sigaction_handler;
    
    sigset_t   sa_mask;
    int        sa_flags;
    void     (*sa_restorer)(void);
};

/* POSIX macros to hide the union implementation: */
#define sa_handler   __sigaction_handler.sa_handler
#define sa_sigaction __sigaction_handler.sa_sigaction
```

### `sa_handler` vs `sa_sigaction`

a signal can only have one execution path, so these two function pointers share the same memory via a `union`, the kernel picks which one to actually call based on `sa_flags`.

`sa_handler` is the legacy callback, just gets the signal number as a single int (good for simple state toggling), and it's also where you'd assign `SIG_DFL`/`SIG_IGN` directly. `sa_sigaction` is the modern one, only gets invoked if `SA_SIGINFO` is set, and gets a lot more: the signal number, a pointer to the `siginfo_t` metadata struct (fault address, sender's PID, RT payloads), and a `void *` you can cast to `ucontext_t` for the raw register state before the interruption.

> this is like asking you how do you want to handle this signal. use `sa_handler` when you write a simple custom function that only needs to know the signal number: the mere arrival of the signal is all the information you need, you don't care about the CPU state or who sent it, you just need to flip a binary state. use `sa_sigaction` when you write an advanced custom function that needs deep kernel metadata (like the exact memory address that caused a segfault, or the PID of the process that sent the signal).

### `sa_mask`

a `sigset_t` bitmask of which other signals get temporarily blocked while this specific handler is executing.

> You tell the kernel, "If this signal arrives while I am busy, put it in the waiting room." It is not destroyed; it is delayed.
> If a signal arrives, the kernel will stop other identical signals so they do not interrupt the handling of the first signal (we don't want the termination process to start over every time we press Ctrl+C). With this mask, we can do this with other signals, too. If we add another signal to this mask, that signal will wait until the handling is done.

mechanically, the kernel just ORs the thread's current blocked mask with `sa_mask` for the duration of the handler. and by default, the signal currently being handled gets added to this mask automatically too, so a `SIGSEGV` handler can't get infinitely re-interrupted by another `SIGSEGV` before it finishes.

```c
struct sigaction sa;

sa.sa_handler = signal_handler;
sigemptyset(&sa.sa_mask);

// Explicitly block BOTH signals during handler execution

sigaddset(&sa.sa_mask, SIGUSR1);
sigaddset(&sa.sa_mask, SIGUSR2);


sigaction(SIGUSR1, &sa, NULL);
sigaction(SIGUSR2, &sa, NULL);
```

### `sa_flags`

a bitmask that changes the kernel's delivery/execution behavior. the ones that matter:

- `SA_SIGINFO`: routes execution to `sa_sigaction` instead of `sa_handler`. (if you're using `sa_sigaction` you have to set this.)
- `SA_RESTART`: auto-restarts interruptible syscalls (like `read()` or `wait()`) after the handler completes, instead of them returning an `EINTR` error.
- `SA_NODEFER`: turns off the implicit self-blocking, lets the handler get preempted by another instance of the exact same signal.
- `SA_ONSTACK`: runs the handler on an alternate memory stack (previously registered via `sigaltstack()`) instead of the normal one, which is what makes it possible to handle a `SIGSEGV` caused by stack overflow in the first place.

### `sa_restorer`

obsolete/internal-only at this point. it used to let user-space provide its own trampoline code to fire the `sigreturn` syscall, but modern Linux just uses the vDSO's `__vdso_rt_sigreturn` automatically, so touching this field manually is both unnecessary and genuinely dangerous. POSIX says applications shouldn't use it.
