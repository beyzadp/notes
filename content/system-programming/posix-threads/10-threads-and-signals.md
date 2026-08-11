---
title: "10. Threads and Signals"
---

part of the [[index|posix-threads]] series.

Handling POSIX signals with threads is primarily about understanding that **signals are a process-level concept with per-thread eligibility rules**. A signal may be *generated for the process*, but the kernel ultimately **delivers it to one thread** that is not blocking it (or to a specific thread if explicitly targeted). Correct designs avoid doing complex work in asynchronous signal handlers and instead route signals to a dedicated "signal thread" using synchronous waiting.

## The POSIX Signal Model in Multithreaded Programs


In POSIX, **each thread has its own signal mask**, but **signal dispositions** (the action for a signal: default/ignore/handler) are **shared across the entire process**. This split is the root of most subtle behaviors: which signals are eligible for which threads is per-thread, but how a signal is handled once delivered is process-wide.

Signals are generated either **for the process** or **for a specific thread**. Process-directed signals include those sent by `kill(pid, sig)`, by the terminal driver (e.g., `SIGINT`), and many kernel-generated signals related to process-wide events. 

Thread-directed signals include those sent via `pthread_kill()` (and in Linux, some signals can be thread-directed by `tgkill` internally).

When a process-directed signal becomes pending, the kernel selects **one thread** to deliver it to among those that do not block it. POSIX intentionally leaves the selection policy somewhat unspecified; implementations commonly choose an arbitrary eligible thread. The important rule is: **only one thread runs the handler for a given delivery**. 

> When a signal (like `SIGINT` from pressing Ctrl+C) is sent to a **process** with 10 threads, the kernel looks at all 10 threads, finds which ones have not blocked that signal, and picks exactly one to "interrupt." That chosen thread temporarily pauses its regular work, runs the handler function, and then goes back to its task. The other 9 threads keep running as if nothing happened.

Signals like `SIGSEGV` (Segmentation Fault) or `SIGFPE` (Divide by Zero) are **synchronous**. They are caused by a specific line of code.

Because the error happened on a specific CPU core executing a specific thread's code, the signal is sent directly to that thread. **Masking these is dangerous.** If you block a `SIGSEGV`, the behavior is undefined, usually the process just crashes immediately because the CPU can't "skip" the broken instruction.

Before a signal is actually handled, it lives in a "pending" state. In multithreaded apps, there are two separate "waiting rooms":

1. **Process Pending:** The signal is waiting for _any_ available thread to take it.
    
2. **Thread Pending:** The signal is specifically waiting for _that_ thread to unblock it.

Standard (non-real-time) signals are not queued: multiple occurrences may coalesce into one pending instance. Real-time signals (`SIGRTMIN`..`SIGRTMAX`) are queued and preserve order, which affects designs that need "counting" semantics.


## Thread Signal Masks (`pthread_sigmask`)


`pthread_sigmask()` controls which signals a given thread is willing to accept. It is the thread-level equivalent of `sigprocmask()`; in a multithreaded program, you should use `pthread_sigmask()` to make the intent explicit and portable in thread contexts.

A typical robust pattern is: **block a set of signals in the initial thread before creating any threads**, so that all subsequently created threads inherit that blocked mask. Then, explicitly choose which thread will handle signals and arrange for it to synchronously wait for them using `sigwait()` (or similar). This prevents signals from being delivered "randomly" to arbitrary worker threads.

Key behavioral points:

- A thread that blocks a signal will not have that signal delivered asynchronously to it; process-directed signals may remain pending until some thread unblocks them, or may be delivered to a different thread that does not block them.
- The signal disposition (handler vs ignore vs default) is shared: installing a handler with `sigaction()` affects all threads, even if only some threads unblock the signal.
- Masks are inherited across `pthread_create()`: the new thread starts with a copy of the creating thread's mask at creation time.

**Common mistakes / pitfalls**

- Blocking signals only in the "signal thread" but leaving worker threads unmasked. This causes process-directed signals to be delivered to workers unpredictably.
- Using asynchronous handlers to do non-async-signal-safe work (e.g., `malloc`, `printf`, `pthread_mutex_lock`), which can deadlock or corrupt state.
- Forgetting that `sigaction()` is process-wide: one thread "fixing" signal handlers affects all threads.


**Minimal example: block signals in all threads; handle in one dedicated thread via `sigwait()`**

```c
#define _POSIX_C_SOURCE 200809L
#include <pthread.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

static void *signal_thread(void *arg) {
    sigset_t *set = arg;
    for (;;) {
        int sig;
        int rc = sigwait(set, &sig);   // synchronous receipt
        if (rc != 0) continue;

        if (sig == SIGTERM || sig == SIGINT) {
            // safe to do normal work here: we're not in an async handler
            fprintf(stderr, "Got signal %d, shutting down...\n", sig);
            exit(0);
        }
    }
    return NULL;
}

int main(void) {
    sigset_t set;
    sigemptyset(&set);
    sigaddset(&set, SIGINT);
    sigaddset(&set, SIGTERM);

    // Block in main before creating other threads => inherited by all.
    if (pthread_sigmask(SIG_BLOCK, &set, NULL) != 0) {
        perror("pthread_sigmask");
        return 1;
    }

    pthread_t tid;
    if (pthread_create(&tid, NULL, signal_thread, &set) != 0) {
        perror("pthread_create");
        return 1;
    }

    // workers...
    for (;;) pause();
}
```

how this code works:

The code provided follows the "Robust Pattern" perfectly:

1. **Setup the Mask:** It creates a "Set" containing `SIGINT` and `SIGTERM`.
2. **Block in Main:** `pthread_sigmask(SIG_BLOCK, &set, NULL)` is called immediately. Now `main` is immune.
3. **Spawn Signal Thread:** The `signal_thread` is created. It inherits the "Blocked" status.
4. **Spawn Workers:** (Implicitly) Any workers created now would also be immune.
5. **The Loop:** The `signal_thread` calls `sigwait(&set, &sig)`.
    - Even though the signal is **blocked**, `sigwait` is allowed to "see" it and pull it out of the pending queue.
    - The signal is "consumed" here. No other thread ever sees it.

now lets look at the signal_thread function closer:

```c
static void *signal_thread(void *arg) {
    sigset_t *set = arg;
```
When this thread starts, it receives the `sigset_t` (the list of signals like `SIGINT` and `SIGTERM`) that you created in `main`. It now knows exactly which signals it is responsible for watching.

```c
for (;;) {
    int sig;
```
This thread has one job and one job only: stay alive for the duration of the program and wait for signals. The variable `int sig` is an empty bucket where the kernel will "drop" the number of the signal that just arrived.

```c
int rc = sigwait(set, &sig);   // synchronous receipt
```

This is the most important line in the whole program.

- **It Pauses:** `sigwait` puts this thread to sleep. It consumes **zero CPU power** while waiting. `sigwait` only moves to the next line of code _after_ a signal arrives. This makes the execution flow predictable, like reading a file.


```c
if (sig == SIGTERM || sig == SIGINT) {
    // safe to do normal work here: we're not in an async handler
    fprintf(stderr, "Got signal %d, shutting down...\n", sig);
    exit(0);
}
```

In a standard signal handler, you are **forbidden** from using `printf` or `fprintf` because they aren't "Async-Signal-Safe" (they use internal locks that could cause a deadlock).

**However, inside this loop, you are safe.** Because `sigwait` is a normal function call in a normal thread context, you aren't "interrupting" yourself. 




## Delivering Signals to Specific Threads (`pthread_kill`)


`pthread_kill()` allows sending a signal to a **specific thread** in the same process, identified by `pthread_t`. This is useful for thread coordination, targeted interruption, or waking a thread that is waiting in `sigwait()`/`sigtimedwait()`.

- `pthread_kill(t, sig)` is thread-directed: it targets that thread, not the process as a whole.
- If `sig == 0`, no signal is sent; the call performs error checking (e.g., "does this thread exist?") without delivery. This is often used as a liveness probe.
- Delivery still respects the target thread's mask: if the thread blocks the signal, it will become pending for that thread until unblocked or synchronously waited.

A practical use is to implement "poke the signal thread" behavior with a real-time signal or `SIGUSR1`, but many designs prefer `eventfd`, pipes, condition variables, or `pthread_cond_signal` instead because signals have process-global side effects and complicated interactions.



## Synchronous Signal Handling (`sigwait`)

signal handler functions installed via `sigaction()` run asynchronously and therefore must obey the **async-signal-safe** restriction set. Synchronous waiting (`sigwait`, `sigwaitinfo`, `sigtimedwait`) avoids these restrictions by delivering signals into a normal thread context.

The core rule for correctness is: the signals waited for by `sigwait()` **must be blocked** in the calling thread (and typically in all threads). When the signal is pending, `sigwait()` returns and provides the signal number. With `sigwaitinfo()`/`sigtimedwait()` you can also obtain a `siginfo_t` with sender PID/UID and additional data (especially valuable for real-time signals).

A common architecture is:

- Block chosen "control signals" (like `SIGTERM`, `SIGINT`, `SIGHUP`) in all threads.
- Start one thread that calls `sigwait()` and translates signals into orderly shutdown/reload actions by setting atomic flags, writing to a pipe, or notifying condition variables.
- Keep fatal synchronous signals (`SIGSEGV`, etc.) on defaults or handle them only for logging/minidumps with extreme care.

Below is a complete, working example that matches the architecture i described:

```c
#define _POSIX_C_SOURCE 200809L
#include <errno.h>
#include <pthread.h>
#include <signal.h>
#include <stdatomic.h>
#include <stdbool.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>

// Global application state (could be put into a struct)
static atomic_bool g_shutdown_requested = false;
static atomic_bool g_reload_requested = false;

static pthread_mutex_t g_mu = PTHREAD_MUTEX_INITIALIZER;
static pthread_cond_t g_cv = PTHREAD_COND_INITIALIZER;

static void request_shutdown(void) {
    atomic_store_explicit(&g_shutdown_requested, true, memory_order_relaxed);

    // Wake threads that might be waiting.
    pthread_mutex_lock(&g_mu);
    pthread_cond_broadcast(&g_cv);
    pthread_mutex_unlock(&g_mu);
}

static void request_reload(void) {
    atomic_store_explicit(&g_reload_requested, true, memory_order_relaxed);

    // Wake threads to notice reload request quickly.
    pthread_mutex_lock(&g_mu);
    pthread_cond_broadcast(&g_cv);
    pthread_mutex_unlock(&g_mu);
}

// Dedicated signal thread: waits synchronously for signals and translates them
// into actions.
static void *signal_thread_main(void *arg) {
    (void)arg;

    sigset_t set;
    sigemptyset(&set);
    sigaddset(&set, SIGINT);
    sigaddset(&set, SIGTERM);
    sigaddset(&set, SIGHUP);

    for (;;) {
        int sig = 0;
        int rc = sigwait(&set, &sig);
        if (rc != 0) {
            // sigwait returns an error number on failure.
            fprintf(stderr, "sigwait failed: %s\n", strerror(rc));
            // Usually safest is to request shutdown if signal handling is
            // broken.
            request_shutdown();
            return NULL;
        }

        switch (sig) {
        case SIGINT:
        case SIGTERM:
            fprintf(stderr, "signal thread: got %s -> shutdown\n",
                    (sig == SIGINT) ? "SIGINT" : "SIGTERM");
            request_shutdown();
            return NULL; // exit signal thread after initiating shutdown

        case SIGHUP:
            fprintf(stderr, "signal thread: got SIGHUP -> reload\n");
            request_reload();
            break;

        default:
            // Should not happen because only these are in the set.
            fprintf(stderr, "signal thread: got unexpected signal %d\n", sig);
            break;
        }
    }
}

// Example worker thread that periodically does work, and responds to
// reload/shutdown.
static void *worker_thread_main(void *arg) {
    long id = (long)arg;

    while (!atomic_load_explicit(&g_shutdown_requested, memory_order_relaxed)) {
        // Simulate doing some work
        fprintf(stdout, "worker %ld: working...\n", id);
        fflush(stdout);

        // Wait up to 1 second, but wake early on shutdown/reload broadcast
        struct timespec ts;
        clock_gettime(CLOCK_REALTIME, &ts);
        ts.tv_sec += 1;

        pthread_mutex_lock(&g_mu);
        (void)pthread_cond_timedwait(&g_cv, &g_mu, &ts);
        pthread_mutex_unlock(&g_mu);

        // Handle reload request (edge-triggered style)
        if (atomic_exchange_explicit(&g_reload_requested, false,
                                     memory_order_relaxed)) {
            fprintf(stdout, "worker %ld: reloading config...\n", id);
            fflush(stdout);
            // TODO: reload configuration here (avoid non-thread-safe global
            // rewrites)
        }
    }

    fprintf(stdout, "worker %ld: shutting down cleanly\n", id);
    fflush(stdout);
    return NULL;
}

int main(void) {
    // 1) Block chosen control signals in this thread (and thus in all
    // subsequently created threads)
    sigset_t set;
    sigemptyset(&set);
    sigaddset(&set, SIGINT);
    sigaddset(&set, SIGTERM);
    sigaddset(&set, SIGHUP);

    if (pthread_sigmask(SIG_BLOCK, &set, NULL) != 0) {
        perror("pthread_sigmask");
        return 1;
    }

    // 2) Start the dedicated signal-waiting thread
    pthread_t sigthr;
    if (pthread_create(&sigthr, NULL, signal_thread_main, NULL) != 0) {
        perror("pthread_create(signal_thread)");
        return 1;
    }

    // 3) Start some worker threads
    enum { NWORKERS = 7 };
    pthread_t workers[NWORKERS];

    for (long i = 0; i < NWORKERS; i++) {
        if (pthread_create(&workers[i], NULL, worker_thread_main, (void *)i) != 0) {
            perror("pthread_create(worker)");
            request_shutdown();
            break;
        }
    }

    // 4) Wait for workers to finish (shutdown triggered by SIGINT/SIGTERM)
    for (int i = 0; i < NWORKERS; i++) {
        pthread_join(workers[i], NULL);
    }

    // Signal thread will exit on SIGINT/SIGTERM; if shutdown requested by other
    // means, you might want to also terminate it (e.g., by sending it a signal
    // it waits on). Here it's fine.
    pthread_join(sigthr, NULL);

    fprintf(stdout, "main: exited normally\n");
    return 0;
}
```
