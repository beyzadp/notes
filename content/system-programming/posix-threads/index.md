---
title: "Programming with POSIX Threads: Concurrency, Mutexes, and Deadlock-Free Design"
---

notes on writing concurrent C programs with the POSIX thread library: what threads actually are, the synchronization primitives pthreads gives you, how to avoid races and deadlocks, and how to debug it all when something goes wrong. split into 13 parts, roughly in teaching order, since later sections lean on earlier ones (mutexes before deadlocks, condition variables before the advanced primitives, and so on). written from a Linux/glibc perspective, so some details are Linux-specific rather than universal POSIX behavior.

## the parts

1. **[[01-introduction-to-threads|Introduction to Threads]]**: concurrency vs. parallelism, threads vs. processes, the tradeoffs multithreading brings, and an overview of the POSIX pthread API.

2. **[[02-thread-management|Thread Management]]**: creating threads (`pthread_create`), terminating them (`pthread_exit`/`pthread_cancel`), joining vs. detaching, thread attributes, and thread identification.

3. **[[03-races-and-critical-sections|Data Races and Critical Sections]]**: what a race condition actually is, atomicity and interleaving, the critical section problem, and invariants on shared state.

4. **[[04-mutexes|Synchronization: Mutexes]]**: initializing and destroying mutexes, locking/unlocking, `trylock`, mutex attributes (recursive, errorcheck), and priority inversion.

5. **[[05-deadlock-free-design|Designing Deadlock-Free Code]]**: the four Coffman conditions, lock ordering and hierarchy, and practical deadlock avoidance strategies.

6. **[[06-condition-variables|Synchronization: Condition Variables]]**: state-based synchronization, waiting and waking (`pthread_cond_wait`/`signal`/`broadcast`), spurious wakeups, and how condition variables relate to mutexes.

7. **[[07-advanced-sync-primitives|Advanced Synchronization Primitives]]**: read-write locks, spinlocks, barriers, and POSIX semaphores (named vs. unnamed, wait/post).

8. **[[08-thread-local-storage|Thread-Specific Data (Thread-Local Storage)]]**: why per-thread state is needed, creating/deleting keys, setting and getting values, and destructor functions.

9. **[[09-cancellation-and-cleanup|Thread Cancellation and Cleanup]]**: deferred vs. asynchronous cancellation, cancellation points, and cleanup handlers (`pthread_cleanup_push`).

10. **[[10-threads-and-signals|Threads and Signals]]**: how POSIX signals behave across threads, thread signal masks, delivering signals to a specific thread, and synchronous handling with `sigwait`.

11. **[[11-process-control-and-scheduling|Threads, Process Control & Scheduling]]**: the `fork()` problem in multithreaded code, `pthread_atfork`, what `exec()` does to other threads, contention scope, scheduling policies (`SCHED_FIFO`/`SCHED_RR`/`SCHED_OTHER`), and thread priorities.

12. **[[12-thread-safe-libraries|Writing Thread-Safe Libraries]]**: reentrancy vs. thread-safety, managing global/static state, and one-time initialization (`pthread_once`).

13. **[[13-debugging-multithreaded-programs|Debugging Multithreaded Programs]]**: detecting race conditions, analyzing deadlocks from stack traces, and tooling (Valgrind/Helgrind, ThreadSanitizer).
