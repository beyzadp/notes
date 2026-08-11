---
title: "9. Thread Cancellation and Cleanup"
---

part of the [[index|posix-threads]] series.

POSIX thread cancellation allows one thread to request another thread's termination using `pthread_cancel`. The main hazard is that termination may occur at a point where invariants are temporarily broken: mutexes held, partially updated shared state, open file descriptors, heap allocations not yet linked into tracking structures, etc. "Safe cancellation" is largely about ensuring cancellation only occurs at known safe points and that cleanup handlers restore invariants and release resources.

Cancellation is cooperative: `pthread_cancel` only posts a cancellation request. The target thread actually acts on it based on its cancellation *state* and *type*, and typically at defined cancellation points.


## Cancellation States and Types (Deferred vs. Asynchronous)

Each thread has a cancellation *state* (enabled/disabled) and a cancellation *type* (deferred/asynchronous). State controls whether the thread can be cancelled at all; type controls *when* it reacts.


- Cancellation state:
  - `PTHREAD_CANCEL_ENABLE`: a pending cancellation request can be acted upon.
  - `PTHREAD_CANCEL_DISABLE`: requests remain pending but are not acted upon until re-enabled.

- Cancellation type:
  - `PTHREAD_CANCEL_DEFERRED`: the default; the thread acts on cancellation only at cancellation points (safe, predictable).
  - `PTHREAD_CANCEL_ASYNCHRONOUS`: the thread may be cancelled at (almost) any instruction boundary (dangerous, hard to make correct).

In practice, `PTHREAD_CANCEL_ASYNCHRONOUS` is rarely safe because it can unwind a thread while it holds internal libc locks, while it has a `mutex` locked, or while it is in the middle of updating shared data structures. Even if cleanup handlers run, there may be no reasonable cleanup that can restore invariants if the thread is interrupted in the middle of a critical sequence.

A common disciplined approach is:

- Keep cancellation **deferred**.
- Temporarily **disable** cancellation while holding locks or manipulating invariants that cannot be safely rolled back.
- Insert explicit cancellation checks using `pthread_testcancel()` at locations where you *know* it's safe to exit.




## Cancellation Points

Under deferred cancellation, the thread checks for pending cancellation when it reaches certain library calls that are defined as cancellation points (e.g., blocking operations and other calls that can "naturally" serve as safe interruption points). Typical examples include `read`, `write` (implementation-dependent in some cases), `accept`, `wait`, `pthread_cond_wait`, `sleep`, and many others defined by POSIX.


The key internal concept is that a cancellation point is a place where the library call will, if cancellation is enabled and a request is pending, start cancellation processing: run cleanup handlers, release internal resources as specified, and terminate the thread as if by `pthread_exit(PTHREAD_CANCELED)`. If the call blocks, the cancellation mechanism must also ensure the thread can be woken and cancelled, which is why cancellation is closely tied to how the runtime/kernel handles thread blocking.

You should treat cancellation points as boundaries where control flow may not return normally. That means code like "lock mutex -> call `read` -> unlock mutex" is risky: if `read` is a cancellation point and the thread is cancelled during `read`, the `mutex` will remain locked unless you installed a cleanup handler.

so if we lock a mutex and read some data from somewhere, it would be a mistake to assume it's safe to cancel there: read is a cancellation point, and if cancelled mid-read, that mutex can remain locked forever.

When you need additional explicit safe points, use:

- `void pthread_testcancel(void);`
  - If cancellation is enabled and pending, it triggers cancellation immediately (like an explicit cancellation point).





## Pushing and Popping Cleanup Handlers (`pthread_cleanup_push`)

Cleanup handlers are the primary POSIX mechanism to make cancellation (and `pthread_exit`) safe with respect to resource release. `pthread_cleanup_push(handler, arg)` registers a function to be called if the thread exits or is cancelled while the handler is still on the cleanup stack. `pthread_cleanup_pop(execute)` removes the handler; if `execute` is nonzero, it runs it immediately.

A subtle but important detail is that `pthread_cleanup_push`/`pthread_cleanup_pop` are typically macros that must appear in the same lexical scope (they may introduce a block). You cannot `goto` or `return` in a way that skips the matching pop without invoking undefined behavior in many implementations. Write code so the push/pop are structurally paired.

If you could see what the compiler sees after the macros expand, it looks something like this:


```c
// What you write:
pthread_cleanup_push(my_handler, arg);
// ... your code ...
pthread_cleanup_pop(1);

// What the compiler actually sees:
{  // <--- Opened by the 'push' macro
    register_handler(my_handler, arg);
    // ... your code ...
    unregister_and_run(1);
}  // <--- Closed by the 'pop' macro
```

that's why you can't return or goto

```c
void* my_thread(void* arg) {
    pthread_mutex_lock(&lock);
    pthread_cleanup_push(unlock_handler, &lock);

    if (some_error) {
        return NULL; // ERROR! You are exiting the function 
                     // without closing the "invisible brace" 
                     // from the push macro.
    }

    pthread_cleanup_pop(1);
    return NULL;
}
```

here is a safe method:

```c
void* my_thread(void* arg) {
    int success = 1;
    pthread_mutex_lock(&lock);
    
    // The "Push" starts the block
    pthread_cleanup_push(unlock_handler, &lock);

    if (check_some_condition() == -1) {
        success = 0; 
        // We don't return here! We let the code flow down to the pop.
    } else {
        do_cancellation_point_work(); 
    }

    // The "Pop" closes the block. 
    // This is the ONLY way out of this section of code.
    pthread_cleanup_pop(1); 

    return success ? (void*)0 : (void*)-1;
}
```



- `pthread_cleanup_push(void (*routine)(void *), void *arg)`
  - `routine`: cleanup function (often `pthread_mutex_unlock`, `free`, or a custom release routine).
  - `arg`: argument passed to `routine`.
- `pthread_cleanup_pop(int execute)`
  - `execute = 0`: just unregister.
  - `execute != 0`: execute now, then unregister.

A canonical safe pattern for a mutex around a cancellation point:

```c
#include <pthread.h>
#include <unistd.h>
#include <errno.h>

static pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;

static void cleanup_unlock(void *arg) {
    pthread_mutex_unlock((pthread_mutex_t *)arg);
}

void *worker(void *arg) {
    (void)arg;

    pthread_mutex_lock(&m);
    pthread_cleanup_push(cleanup_unlock, &m);

    // Cancellation point: if cancelled here, cleanup_unlock runs.
    char buf[128];
    ssize_t n = read(STDIN_FILENO, buf, sizeof(buf));

    // Normal path: pop and execute unlock.
    pthread_cleanup_pop(1);

    if (n < 0) {
        // handle error
    }
    return NULL;
}
```

This works because the mutex unlock is registered *before* reaching the cancellation point. If cancellation occurs during `read`, the runtime unwinds the cleanup stack and calls `cleanup_unlock`, preventing a permanent lock leak. On normal completion, `pthread_cleanup_pop(1)` runs the unlock explicitly.

A common refinement is to disable cancellation in very small critical sections and then re-enable it immediately after installing cleanup handlers or exiting the critical region, keeping the thread responsive to cancellation without sacrificing correctness.
