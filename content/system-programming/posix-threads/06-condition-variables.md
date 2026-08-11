---
title: "6. Synchronization: Condition Variables"
---

part of the [[index|posix-threads]] series.

Condition variables provide **state-based blocking**: a thread can sleep until some shared program state (e.g., "queue not empty", "buffer has space", "work is available", "all workers finished") becomes true. Unlike busy-waiting, this yields the CPU while waiting; unlike a mutex alone, it gives a structured way to sleep until a state transition occurs. In pthreads, a condition variable is always used **together with a mutex** that protects the shared state used in the predicate (the "condition").

A condition variable does **not** "store a condition" internally; it is a **waiting queue + signaling mechanism**. The *actual condition* is a predicate over shared variables, and correctness depends on: (1) protecting those variables with a mutex, and (2) checking the predicate in a loop.

## State-Based Synchronization

State-based synchronization means threads coordinate on **changes to shared variables**, not merely on mutual exclusion. The pattern is:

1. A mutex protects shared state.
2. A predicate over that state defines when progress is allowed (e.g., `count > 0`).
3. When the predicate is false, a thread waits on a condition variable, releasing the mutex and sleeping.
4. Another thread changes the state and signals/broadcasts to wake waiters so they can re-check the predicate.

Internally, this works because `pthread_cond_wait` provides an **atomic "release mutex + park thread"** operation with respect to the condition variable, preventing a classic lost-wakeup window where a signal occurs between "check predicate" and "go to sleep".

There are two distinct roles a thread can play:

- **Waiter**: needs some predicate to become true; holds the mutex, checks the predicate, and waits if false.
- **Signaler**: changes the shared state under the mutex such that the predicate may become true, then signals/broadcasts.

Key invariants and risks:

- The predicate must be evaluated while holding the same mutex that is used with `pthread_cond_wait`. If you read state without the mutex, you can observe inconsistent or stale values and either wait incorrectly or proceed incorrectly.
- Signaling is not queued as "events" in a reliable way. Signaling when no one waits is often fine, but you must still write code so that a later waiter won't wait forever; that is achieved by **state + predicate**, not by relying on "remembered signals".

## Initialization and Destruction


A `pthread_cond_t` may be initialized statically or dynamically. It can also be configured via a `pthread_condattr_t` (notably to choose a clock for timed waits).

Common initialization approaches:

- Static initialization with `PTHREAD_COND_INITIALIZER`.
- Dynamic initialization with `pthread_cond_init`.

Similarly, destruction uses `pthread_cond_destroy`. Destruction requires that no other thread is currently waiting on or using the condition variable, otherwise behavior is undefined.

**API behaviors / rules:**

- `int pthread_cond_init(pthread_cond_t *cond, const pthread_condattr_t *attr)`
  - `attr == NULL` selects default attributes.
  - Returns 0 on success, error number on failure.
- `int pthread_cond_destroy(pthread_cond_t *cond)`
  - Requires no waiters and no concurrent operations.
  - Returns 0 on success, error number on failure (commonly `EBUSY` if there are waiters on some implementations).

**Pitfalls:**

- Destroying a condition variable while a thread may still be inside `pthread_cond_wait` (or about to wait) is a race leading to undefined behavior.
- Reinitializing a condition variable in place without synchronization can strand waiters on an object that is no longer valid.

## Waiting on a Condition (`pthread_cond_wait`)

`pthread_cond_wait` is the core primitive: it allows a thread to sleep until signaled, but only safely when paired with a mutex and a predicate.

The correct structure:

1. Lock the mutex protecting the shared state.
2. While the predicate is false, call `pthread_cond_wait(&cond, &mutex)`.
3. On return, the mutex is locked again; re-check predicate; proceed if true.
4. Unlock the mutex.

`pthread_cond_wait` must be understood as an atomic sequence with respect to the mutex and the condition variable's wait queue.

**Step-by-step internal mechanics (conceptual):**

- The calling thread must hold `mutex`.
- The implementation:
  - Enqueues the thread onto `cond`'s wait set.
  - Atomically releases `mutex` and blocks the thread (parks it).
- When the thread is woken (by `signal`, `broadcast`, or spuriously):
  - It attempts to re-acquire `mutex` before returning to user code.
  - Only after it holds `mutex` again does `pthread_cond_wait` return.

This "unlock + sleep" atomicity is what prevents lost wakeups between the time you decide to sleep and the time you actually block.

**API behaviors / rules:**

- `int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex)`
  - The caller must hold `*mutex` on entry.
  - On return (success or certain failures), `*mutex` is held by the caller.
  - Returns 0 on success, error number on failure.

**Pitfalls:**

- Calling `pthread_cond_wait` without holding the mutex: undefined behavior.
- Checking the predicate without the mutex, then waiting: can miss state transitions and sleep indefinitely.
- Assuming that returning from `pthread_cond_wait` implies the predicate is true: it may be false (spurious wakeup, or another thread consumed the resource first).

There's also a timed variant: `pthread_cond_timedwait(cond, mutex, abstime)` behaves the same way, but returns `ETIMEDOUT` if `abstime` (an absolute time, not a duration) passes before being signaled. The mutex is still reacquired before returning even on timeout, and you must still re-check the predicate afterward, a timeout is not proof the predicate became true.

## Waking Up Threads (`pthread_cond_signal`, `pthread_cond_broadcast`)

Signaling wakes threads currently blocked in `pthread_cond_wait` on the same condition variable. Importantly, signaling does not "grant" the mutex; it merely makes waiters runnable. A woken waiter still must re-acquire the mutex before returning from `pthread_cond_wait`. This means that the thread that signaled may continue holding the mutex and can run to completion of its critical section before any woken waiter makes progress.

**API behaviors / rules:**

- `int pthread_cond_signal(pthread_cond_t *cond)`
  - Wakes *at least one* waiting thread (if any).
  - If no threads are waiting, it has no effect.
- `int pthread_cond_broadcast(pthread_cond_t *cond)`
  - Wakes *all* waiting threads (if any).
  - If no threads are waiting, it has no effect.

**When to use `signal` vs `broadcast`:**

- Use `pthread_cond_signal` when a single state change can enable **only one** waiter to make progress (e.g., adding a single item to a queue).
- Use `pthread_cond_broadcast` when a state change can enable **many/all** waiters (e.g., "shutdown now", "configuration updated", "barrier released", "space became available for multiple items"), or when it's difficult to know how many waiters are enabled safely.

**Pitfalls / subtle behaviors:**

- Signaling without holding the mutex is not always *formally* forbidden by POSIX, but it is a common source of races because the state change and the wakeup are no longer grouped. The robust pattern is: **lock -> change state -> signal/broadcast -> unlock**.
- Broadcasting can cause a "thundering herd": many threads wake, contend for the mutex, and most go back to sleep after seeing the predicate still false. This is correct but can be a performance issue.
- Waking order is not guaranteed. You cannot rely on FIFO fairness among waiters.

**Example: bounded queue (signal on transitions)**

```c
#include <pthread.h>
#include <stdlib.h>

typedef struct {
    int *buf;
    size_t cap, head, tail, count;
    pthread_mutex_t m;
    pthread_cond_t not_empty;
    pthread_cond_t not_full;
} bq_t;

void bq_init(bq_t *q, size_t cap) {
    q->buf = malloc(cap * sizeof(int));
    q->cap = cap; q->head = q->tail = q->count = 0;
    pthread_mutex_init(&q->m, NULL);
    pthread_cond_init(&q->not_empty, NULL);
    pthread_cond_init(&q->not_full, NULL);
}

void bq_put(bq_t *q, int x) {
    pthread_mutex_lock(&q->m);
    while (q->count == q->cap) {
        pthread_cond_wait(&q->not_full, &q->m);
    }
    q->buf[q->tail] = x;
    q->tail = (q->tail + 1) % q->cap;
    q->count++;
    pthread_cond_signal(&q->not_empty); // at least one item exists now
    pthread_mutex_unlock(&q->m);
}

int bq_get(bq_t *q) {
    pthread_mutex_lock(&q->m);
    while (q->count == 0) {
        pthread_cond_wait(&q->not_empty, &q->m);
    }
    int x = q->buf[q->head];
    q->head = (q->head + 1) % q->cap;
    q->count--;
    pthread_cond_signal(&q->not_full); // at least one slot exists now
    pthread_mutex_unlock(&q->m);
    return x;
}
```

the state (`count`, `head`, `tail`) is protected by a mutex, and each wait is tied to a predicate on that state. Each state transition that could unblock a waiter issues the corresponding `signal`.







## Handling Spurious Wakeups

A spurious wakeup means `pthread_cond_wait` can return even though no thread called `pthread_cond_signal`/`pthread_cond_broadcast`, or even if the predicate is still false. Additionally, even with a "real" wakeup, the predicate may be false because another thread acquired the mutex first and consumed the resource.

There are two main reasons a thread might wake up without any signal or even if the predicate is still false:

**1. POSIX Signals and System Call Interruption**

Under the hood, `pthread_cond_wait` relies on blocking system calls (like `futex` on Linux, a kernel primitive for parking and waking threads, more on it in the semaphores section).

If the thread is sleeping on a `futex` and the process receives an unmasked POSIX signal (like `SIGALRM` or `SIGCHLD`), the kernel interrupts the sleeping system call to handle the signal. The system call returns an error like `EINTR` (Interrupted system call).

If the pthreads library were to secretly catch the `EINTR` and go back to sleep without returning control to your application, it could miss a legitimate signal. Here is the step-by-step race condition of how that disaster would happen.

Imagine the pthreads library _is_ designed to automatically swallow `EINTR` and go back to sleep. Here is how it would break your program:

1. **Thread A (Consumer)** is sleeping inside `pthread_cond_wait`, waiting for data. It does not currently hold the mutex (the wait function releases it).

2. **A POSIX Signal (like a timer)** hits the process. The kernel interrupts Thread A's sleep.

3. **Thread A wakes up inside the pthreads library** with an `EINTR` error. It is momentarily awake, but it hasn't re-acquired the mutex yet. It is preparing to automatically go back to sleep.

4. **Thread B (Producer)** suddenly gets scheduled on another CPU core. It locks the mutex, adds data to the queue (making your predicate true!), calls `pthread_cond_signal`, and unlocks the mutex.

5. **The Signal goes into the void.** Thread B sent the wakeup signal, but Thread A wasn't technically sleeping on the condition variable at that exact microsecond, it was awake inside the library handling the `EINTR`.

6. **Thread A goes back to sleep.** Following its "automatic restart" rule, Thread A blindly goes back to sleep on the condition variable.

**The Result:** Thread A is now asleep forever. Thread B already sent the signal and assumes Thread A is processing the data. Your program is deadlocked.

**2. Multiprocessor Broadcast Inefficiencies**

When `pthread_cond_broadcast` is called, multiple threads need to be moved from the condition variable's wait queue to the mutex's wait queue. On multi-core systems, ensuring that _only_ the exact intended threads wake up, and that they wake up perfectly synchronized with the mutex state, requires locking internal pthreads data structures.

If POSIX mandated zero spurious wakeups, every single condition variable operation would require heavy, global locks across all CPU cores. By allowing spurious wakeups, the OS can use lock-free or highly granular locking mechanisms, making the 99% of normal, non-waking condition checks vastly faster.


> Therefore, the predicate must always be checked in a `while` loop, not an `if`.

- Correct code must treat `pthread_cond_wait` as: "sleep **until woken**, then re-check predicate under mutex".
- The only reliable indicator that you can proceed is that the predicate is true when tested while holding the mutex.

- Using `if (predicate_false) pthread_cond_wait(...)` can lead to:
  - proceeding with predicate false (memory safety bugs, underflow, invariant violations),
  - deadlocks (thread assumes it was "really signaled" and fails to wait again),
  - rare timing-dependent failures that are hard to reproduce.



## Condition Variables vs. Mutexes

A mutex provides **mutual exclusion**: it serializes access to shared state so updates are atomic with respect to other threads. However, a mutex alone does not provide a way to efficiently wait for a *future* state; without a condition variable you typically end up with either polling (busy-waiting) or ad-hoc sleep loops that introduce latency and races.

A condition variable provides **blocking until a state transition**, but it does not protect the state itself. In pthreads, the condition variable is useless without a mutex because the wakeup must be tied to consistent observation and modification of the predicate state.

**Correct usage pattern (summary):**

- Waiter:
  - Lock mutex
  - `while (!predicate) pthread_cond_wait(&cv, &mutex);`
  - Perform action while predicate true
  - Unlock mutex
- Signaler:
  - Lock mutex
  - Update shared state to make predicate true (or more true)
  - `pthread_cond_signal`/`pthread_cond_broadcast`
  - Unlock mutex

This division of responsibilities ensures correctness under weak scheduling guarantees, handles spurious wakeups, and prevents missed state transitions.
