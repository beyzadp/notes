---
title: "11. Threads, Process Control & Scheduling"
---

part of the [[index|posix-threads]] series.

## Threads and Process Control

Threading complicates classic UNIX process control because `fork()` copies a process in a way that is only fully coherent in single-threaded execution. The central hazard is that in the child after `fork()`, only one thread exists, but the memory image includes the "frozen" state of synchronization primitives that may have been held by threads that no longer exist.

### The `fork()` Problem in Multithreaded Code

When a multithreaded process calls `fork()`, **only the calling thread is duplicated into the child**. All other threads vanish. However, the child inherits the entire address space snapshot, including libc internals and user-space locks in whatever state they were at the instant of fork. If any other thread held a `pthread_mutex_t` (or a malloc arena lock, stdio lock, etc.), the child can be left with a permanently locked object that can never be unlocked, since the owning thread is gone. 

This shows up as deadlocks in the child when it tries to do almost anything nontrivial, especially:

- calling `malloc`/`free` (allocator locks),
- calling `printf`/`fprintf` (stdio locks),
- using `pthread_mutex_lock` on a mutex that was held at fork time.

POSIX therefore restricts what is safe to do in the child between `fork()` and `exec()` in a multithreaded program. The safe set is essentially: **async-signal-safe functions only** (similar to signal handlers). In practice, the intended model is "`fork()` then immediately `exec()`". 

A safe pattern is:

- Parent: prepare arguments and environment.
- `fork()`.
- Child: call only async-signal-safe functions, then `execve()`. If `execve` fails, call `_exit()` (not `exit()`) to avoid running atexit handlers and flushing stdio (which may lock).

```c
pid_t pid = fork();
if (pid == 0) {
    // Child: async-signal-safe only
    execve(path, argv, envp);
    _exit(127); // exec failed
}
// Parent continues
```
By calling `exec()` immediately after `fork()`, you throw away the entire dangerous, corrupted memory state inherited from the multithreaded parent.

The child process begins executing the new program with a clean slate, entirely side-stepping the abandoned lock problem.


### Fork Handlers (`pthread_atfork`)



`pthread_atfork()` registers callbacks to run around `fork()` to improve consistency by acquiring and releasing locks in a controlled manner. It takes three handlers:

- `prepare`: called in the parent **before** `fork()`; typically locks global mutexes to prevent concurrent mutation during the fork snapshot.
- `parent`: called in the parent **after** `fork()`; typically unlocks what `prepare` locked.
- `child`: called in the child **after** `fork()`; typically reinitializes or unlocks locks to a safe state for the single-threaded child.

This mechanism is most useful when you maintain your own global locks and need to ensure the child won't inherit them in a locked state. It does not magically make all libraries safe; libc itself uses internal `atfork` handlers to try to keep parts of its state coherent, but you still must adhere to the "fork then exec quickly" discipline if the program is meaningfully multithreaded.

Correct usage requires extremely careful lock ordering. If `prepare` locks multiple mutexes, it must do so in a globally consistent order to avoid deadlocking with other code that locks them in a different order.




A minimal sketch:

```c
pthread_mutex_t g_lock = PTHREAD_MUTEX_INITIALIZER;

static void prepare(void) { pthread_mutex_lock(&g_lock); }
static void parent(void)  { pthread_mutex_unlock(&g_lock); }
static void child(void)   { pthread_mutex_unlock(&g_lock); } // or reinit if needed

int main(void) {
    pthread_atfork(prepare, parent, child);
    // ...
}
```

Note: "unlock in child" is only safe if you know `g_lock` was locked by `prepare` in the forking thread. For more complex states, the child handler may need to `pthread_mutex_init` into a known unlocked state (but reinitializing robustly has its own constraints if other references exist).







### `exec()` and Thread Termination


An `exec*()` call replaces the current process image with a new program. This has a clean interaction with threads: after a successful `exec`, **the new program starts fresh**, and the previous process's threads no longer exist. Practically, the thread that called `exec` is the one that transitions into the new program image; everything else is gone because the entire address space and runtime are replaced.

This is why the recommended approach for multithreaded programs that need to spawn a new program is: `fork()` and then `exec()` immediately in the child. `exec()` discards the inconsistent inherited thread library state and synchronization state, because the entire userspace is reinitialized.

Subtle but important details:

- File descriptors remain open across `exec` unless marked `FD_CLOEXEC` (or opened with `O_CLOEXEC`). Threading makes it easier to accidentally leak descriptors into exec'd children if you don't consistently use close-on-exec.
- Signal dispositions: dispositions set to `SIG_IGN` are typically preserved across `exec`, while dispositions set to a custom handler are reset to default in the new program (POSIX specifies resets for caught signals). The signal mask is preserved across `exec` unless changed, so be careful: if you blocked signals process-wide for a signal-thread design, you may need to unblock/reset in the exec'd program or before `exec` depending on intent.
- Only async-signal-safe functions should run between `fork` and `exec` in a multithreaded parent; `exec` is the operation that "cleans the slate".

## Thread Scheduling and Real-Time Execution

Modern OS schedulers multiplex CPU time across runnable threads. You usually have way more programs and threads running than you have physical CPU cores. The OS acts like a hyperactive traffic cop ("the scheduler"), rapidly switching the CPU's attention between all these threads so fast that they appear to be running at the exact same time. This is "multiplexing."

On Linux with pthreads, "thread scheduling" typically means kernel scheduling of *kernel threads* (each `pthread_t` maps to a schedulable entity). For real-time behavior, you mainly control (1) the scheduling *policy* (`SCHED_OTHER`, `SCHED_FIFO`, `SCHED_RR`) and (2) the scheduling *parameters* (notably priority for real-time policies).

What you can actually achieve is constrained by permissions (usually `CAP_SYS_NICE`), CPU availability, and the fact that real-time scheduling guarantees are about *eligibility to run*, not "your work completes by a deadline" unless the whole system is engineered for it (bounded interrupts, bounded critical sections, no unbounded blocking in kernel paths, etc.).

### Contention Scope (System vs. Process)

Whether threads compete for CPU time against all system threads or just threads within the same process.

POSIX defines a "contention scope" attribute for threads: `PTHREAD_SCOPE_SYSTEM` vs `PTHREAD_SCOPE_PROCESS`. 

- **`PTHREAD_SCOPE_PROCESS` (Local Competition):** In this model, the OS kernel only sees your overall program (the process). It gives your program a chunk of CPU time. Then, a manager _inside_ your program decides how to divvy up that time among your specific threads. Your threads only compete with their "siblings."
    
- **`PTHREAD_SCOPE_SYSTEM` (Global Competition):** In this model, the OS kernel sees every single thread individually. Your threads skip the local manager and compete directly with every other thread running on the entire computer (like your web browser, background system updates, or Spotify).

On Linux (NPTL), threads are 1:1 kernel-scheduled, so `PTHREAD_SCOPE_SYSTEM` is effectively the only meaningful mode, and `PTHREAD_SCOPE_PROCESS` is typically *unsupported* (attempting to set it often yields `ENOTSUP`). This matters because it tells you that any fairness/priority decisions are made by the kernel scheduler across *all* runnable threads; you cannot rely on a user-level thread library to time-slice threads inside a process if the kernel schedules only one "process entity."

Practically, when you change a thread's policy/priority on Linux, you're changing how it competes against *every other runnable thread* on the machine, including threads from other processes, hence the need for privilege and the possibility of starving unrelated work.


### Scheduling Policies (`SCHED_FIFO`, `SCHED_RR`, `SCHED_OTHER`)

Choosing between standard time-sharing, real-time First-In-First-Out, or real-time Round-Robin scheduling.

1. `SCHED_OTHER`: The Everyday Citizen (Time-Sharing)

- This is the default setting for 99.9% of programs you run.
- It uses the Linux CFS (Completely Fair Scheduler). The OS tries to be a perfectly fair parent, slicing up CPU time so every program gets a turn.

- **Priority:** It doesn't use standard real-time priorities. Instead, it uses "nice" values. If you make a thread "nicer," it politely lets other `SCHED_OTHER` threads have more CPU time.

- **The Catch:** There is no guaranteed order. The OS is constantly shuffling these threads around based on load, who woke up recently, and who has been waiting the longest.


2. `SCHED_FIFO` (First-In, First-Out): The Dictator

- A strict, real-time policy for absolute VIPs.
- **How it works:** If a `SCHED_FIFO` thread is ready to run, it immediately kicks any `SCHED_OTHER` thread off the CPU.

- **The Catch (No Time-Slicing):** If you have two `SCHED_FIFO` threads with the exact same priority, the first one to grab the CPU will keep it _forever_. It will not share. It only stops running if it finishes, voluntarily pauses (yields), asks for data (blocks), or is interrupted by an even higher-priority VIP.
    
- if you write an infinite loop in a high-priority `SCHED_FIFO` thread, it will "starve" the system. The OS will never get CPU time to run your mouse, your keyboard, or your screen updates. Your computer will completely freeze.


3. `SCHED_RR` (Round-Robin): The Cooperative VIPs

- Another real-time policy, identical to `SCHED_FIFO` in its power to dominate normal programs, but with built-in sharing for its peers.

- **How it works:** It adds a "time slice" (time quantum). If you have three `SCHED_RR` threads running at the _exact same_ priority level, the OS forces them to pass the baton. Thread A gets a few milliseconds, then Thread B, then Thread C, then back to Thread A.

- **The Benefit:** It prevents one VIP thread from accidentally starving other VIP threads of the exact same rank.

Key rules/guarantees (Linux/POSIX-style behavior):
- Real-time policies (`SCHED_FIFO`, `SCHED_RR`) always outrank `SCHED_OTHER` when runnable.
- Higher `sched_priority` outranks lower (within real-time classes).
- `SCHED_FIFO` has no timeslice at a given priority; `SCHED_RR` does.
- A "real-time" policy does **not** guarantee deadlines; it guarantees scheduling precedence given runnability.

### Modifying Thread Priorities

In pthreads, you typically control scheduling using `pthread_setschedparam()` (for an existing thread) or via thread attributes (`pthread_attr_setschedpolicy`, `pthread_attr_setschedparam`) before creating a thread. Internally, on Linux this maps to kernel scheduler attributes; attempting to raise a thread into real-time class or to a higher real-time priority generally requires privileges (`CAP_SYS_NICE`) or an `rlimit` configuration allowing RT priorities.

Important behavioral points:
- `SCHED_FIFO`/`SCHED_RR` use `struct sched_param.sched_priority` (range queried via `sched_get_priority_min/max(policy)`).
- `SCHED_OTHER` ignores `sched_priority` on Linux; "priority-ish" control is via `nice`/CFS weights.
- Thread scheduling attributes may be *inherited* at create time unless you set `PTHREAD_EXPLICIT_SCHED`.

Common mistakes/pitfalls:
- Setting policy/priority in the `pthread_attr_t` but forgetting `pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED)`, resulting in the new thread inheriting the creator's scheduling instead.
- Creating a `SCHED_FIFO` thread that does CPU work without blocking/yielding, leading to starvation.
- Using real-time priority while holding locks or doing I/O, causing priority inversion or long non-preemptible sections.

here is a real life scenario:

In professional Linux audio software like PipeWire or JACK, processing live sound requires strict microsecond timing to prevent audio buffers from emptying and causing loud pops or stutters. To achieve this, the application elevates its critical audio-processing thread to the real-time `SCHED_FIFO` policy, ensuring it instantly preempts normal applications for CPU time whenever an audio chunk needs processing. Because Linux threads compete globally against the entire system, granting this VIP status is dangerous and usually requires administrator privileges, so desktop Linux uses a secure background service called `rtkit` to safely grant these real-time permissions on the fly without giving the app full control over your machine.
