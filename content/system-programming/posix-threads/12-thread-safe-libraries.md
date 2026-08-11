---
title: "12. Writing Thread-Safe Libraries"
---

part of the [[index|posix-threads]] series.

A library is "thread-safe" when its public API functions can be called simultaneously from multiple threads *without causing data races or corrupting internal state*. This requires careful control of shared state, correct synchronization, and well-defined behavior under concurrency (including what happens under cancellation, signals, forks, and during process teardown). You also want to avoid forcing unnecessary serialization: a library that puts a single global mutex around everything may be correct but can destroy scalability.





## Reentrancy vs. Thread-Safety


Functions that can be safely interrupted and re-entered versus functions protected by internal locks.

**Reentrancy** is stronger and more specific than thread-safety.  A reentrant function can be safely called while a previous invocation is "in progress," including if interrupted by a signal handler that calls the same function. This implies the function cannot rely on shared mutable state without strict discipline; it typically must use only stack-local state, immutable globals, or atomic operations that are async-signal-safe (and those are extremely limited in POSIX).

**Thread-safety** usually means concurrent calls from multiple threads won't cause undefined behavior. Many thread-safe functions are *not reentrant* because they rely on internal locks and non-async-signal-safe primitives. For example, a function that uses `malloc` and a `pthread_mutex_t` internally might be thread-safe, but calling it from a signal handler can deadlock or corrupt state because `malloc` and mutex locking are not async-signal-safe.

A useful mental model:  
- Reentrant usually implies thread-safe, but not always if it uses unsafely shared resources.
- Thread-safe does not imply reentrant (internal locks break async-signal-safety).

Common pitfalls:
- Using internal mutexes and then calling the API from a signal handler (can deadlock).
- Returning pointers to internal static buffers (thread-unsafe and non-reentrant).
- Hiding global state behind locks but then invoking user callbacks while holding those locks, creating deadlock/re-entrancy hazards.


here is a Thread-Safe, but NOT Reentrant function:

```c
#include <pthread.h>

pthread_mutex_t my_lock = PTHREAD_MUTEX_INITIALIZER;
int shared_counter = 0;

// Thread-Safe (locks protect the data) but NON-REENTRANT
void do_work() {
    pthread_mutex_lock(&my_lock);  // 1. Thread A locks the door
    
    // ---> IMAGINE THE OS INTERRUPTS THREAD A RIGHT HERE <---
    
    shared_counter++;              
    
    pthread_mutex_unlock(&my_lock); 
}

// The OS jumps here if the user presses Ctrl+C
void signal_handler(int signum) {
    // DEADLOCK! 
    // The handler tries to lock the door, but Thread A already locked it.
    // Thread A can't unlock it because it's paused waiting for this handler to finish.
    do_work(); 
}
```


and a reentrant function:

```c
// REENTRANT and Thread-Safe
// No locks, no global variables. Safe to interrupt at any millisecond.
int do_safe_work(int input_number) {
    
    // local_result is created fresh on the stack every single time 
    // this function is called, even by a signal handler.
    int local_result = input_number * 2; 
    
    return local_result;
}

void signal_handler(int signum) {
    // PERFECTLY SAFE!
    // It gets its own isolated 'local_result' variable.
    int urgent_math = do_safe_work(50); 
}
```

this is also thread safe.

for example, the `rand_r()` function is a **reentrant** version of `rand()`.

## Managing Global and Static Variables

Global/static mutable variables are the main source of unintended sharing in libraries. In C, any non-`const` file-scope or `static` function-scope object is shared across all threads in the process. To be thread-safe, accesses must be synchronized so there is no data race, and invariants are preserved.

There are three common strategies:

1) **Eliminate shared mutable state** by making state explicit. Instead of hidden globals, expose an opaque "context" handle allocated per instance (or per caller) and pass it to all functions. This is the most scalable and testable approach, and avoids cross-client interference.

2) **Protect shared state with synchronization** such as `pthread_mutex_t`, `pthread_rwlock_t`, or atomics. This works when shared caching or singleton resources are required. You must define which lock protects which variables, document lock ordering rules to avoid deadlocks, and ensure error paths unlock correctly.

3) **Use thread-local storage (TLS)** for per-thread caches or error buffers. TLS avoids locking for per-thread data, but it changes semantics (state becomes per-thread, not global), complicates cleanup (destructors), and may interact with `fork()` or thread exit.

## One-Time Initialization (`pthread_once`)


Ensuring an initialization routine runs exactly once, regardless of how many threads call it simultaneously.

`pthread_once` is the standard pthreads primitive for thread-safe one-time initialization. You define a `pthread_once_t` control object initialized to `PTHREAD_ONCE_INIT`, and any thread that calls `pthread_once(&once, init_fn)` will either run `init_fn` (exactly one thread) or wait until another thread has completed it. After completion, all threads observe the effects.

Internally, typical implementations use a small state machine plus atomic operations and futex-like blocking to ensure (1) only one thread executes the init routine, (2) other threads block efficiently, and (3) there is a memory-ordering guarantee so that writes performed by `init_fn` are visible to threads after `pthread_once` returns.

Rules/guarantees:
- `init_routine` is executed at most once.
- If multiple threads call `pthread_once` concurrently, exactly one executes the routine; others wait.
- After `pthread_once` returns, the initialization's memory effects are visible to the caller (acts like an acquire/release synchronization point).

Common mistakes/pitfalls:
- Calling `pthread_once` recursively where the init routine itself calls `pthread_once` on the same control object (can deadlock).
- Doing complex initialization that can fail, without a defined failure strategy. `pthread_once` has no built-in "retry on failure" mechanism; if the routine sets partial state and returns, callers may see a broken singleton.
- Combining `pthread_once` with `fork()` incorrectly: in a multi-threaded process, calling `fork()` duplicates only the calling thread in the child. If another thread was "in the middle of initialization" holding internal locks, the child can deadlock. For libraries, this is why `pthread_atfork` handlers or "fork-safety" constraints matter.

A standard singleton-init pattern:

```c
#include <pthread.h>
#include <stdlib.h>

static pthread_once_t once = PTHREAD_ONCE_INIT;
static pthread_mutex_t global_lock;
static int *global_table;

static void init_library(void) {
    pthread_mutex_init(&global_lock, NULL);
    global_table = malloc(1024 * sizeof(int));
    // In production, handle malloc failure in a defined way.
}

int lib_do_work(int idx) {
    pthread_once(&once, init_library);

    pthread_mutex_lock(&global_lock);
    int v = global_table[idx];
    pthread_mutex_unlock(&global_lock);

    return v;
}
```

Why this works: all threads calling `lib_do_work` will see `global_lock` and `global_table` initialized exactly once, without races. The mutex then protects accesses to shared state.
