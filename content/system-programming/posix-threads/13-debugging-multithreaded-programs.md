---
title: "13. Debugging Multithreaded Programs"
---

part of the [[index|posix-threads]] series.

Debugging concurrent programs fundamentally differs from debugging sequential code because concurrency bugs are inherently nondeterministic. The same program with identical inputs can produce different outcomes across runs due to variations in thread scheduling, making traditional print-based debugging and reproducing techniques largely ineffective. Understanding how to systematically identify race conditions, analyze deadlocks, and leverage dynamic analysis tools is essential for building reliable multithreaded systems.

The core challenge lies in the interleaving of operations across threads. A program may execute correctly millions of times, only to fail when the scheduler happens to produce a particular ordering that exposes a latent bug. This temporal sensitivity means that debugging isn't about finding where the code is "wrong" in a static sense, but rather about identifying where the programmer's mental model of execution order doesn't match the possible actual orderings permitted by the memory model and scheduler.

## Detecting Race Conditions

Detecting race conditions manually requires systematic code analysis. The reviewer must identify all shared state (global variables, heap allocations accessible from multiple threads, and static storage) and then trace every code path that could access this state from multiple threads. The analysis must verify that every such access is protected by a consistent lock, or that synchronization primitives correctly establish ordering.

Some race conditions involve memory ordering. Even when using atomic operations, incorrect memory ordering can cause races on other variables. For example:

```c
int ready = 0;
int data = 0;

void* producer(void* arg) {
    data = 42;
    ready = 1;  // May be reordered before data = 42 without proper ordering
    return NULL;
}

void* consumer(void* arg) {
    while (!ready);  // Spin wait
    printf("data = %d\n", data);  // May print 0 on weakly-ordered architectures
    return NULL;
}
```

Without proper memory barriers or the use of atomic variables with appropriate memory ordering constraints (`memory_order_release` and `memory_order_acquire`), the compiler and CPU are permitted to reorder the stores, causing the consumer to observe `ready == 1` while `data` still reads as 0.



## Analyzing Deadlocks

i have a program which is going to deadlock:

```c
#define _GNU_SOURCE
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define N 5

pthread_mutex_t forks[N];
pthread_barrier_t start_barrier;

void *philosopher(void *arg) {
    int id = *(int *)arg;
    int left = id;
    int right = (id + 1) % N;

    // Synchronize start so they race in the same phase
    pthread_barrier_wait(&start_barrier);

    // Step 1: everyone grabs left
    pthread_mutex_lock(&forks[left]);
    printf("Philosopher %d picked up LEFT fork %d\n", id, left);
    fflush(stdout);

    // Step 2: give others time to grab their left too
    usleep(200 * 1000); // 200ms; adjust as needed

    // Step 3: now everyone tries to grab right -> deadlock
    printf("Philosopher %d trying to pick up RIGHT fork %d\n", id, right);
    fflush(stdout);
    pthread_mutex_lock(&forks[right]); // blocks forever in deadlock

    // Unreachable in deadlock scenario
    printf("Philosopher %d picked up RIGHT fork %d\n", id, right);
    pthread_mutex_unlock(&forks[right]);
    pthread_mutex_unlock(&forks[left]);
    return NULL;
}

int main(void) {
    pthread_t philosophers[N];
    int ids[N];

    for (int i = 0; i < N; i++) {
        pthread_mutex_init(&forks[i], NULL);
    }

    // Barrier releases all philosophers at the same time
    pthread_barrier_init(&start_barrier, NULL, N);

    for (int i = 0; i < N; i++) {
        ids[i] = i;
        pthread_create(&philosophers[i], NULL, philosopher, &ids[i]);
    }

    // Keep process alive; it will hang here once deadlocked (expected)
    for (int i = 0; i < N; i++) {
        pthread_join(philosophers[i], NULL);
    }

    return 0;
}
```

- `pthread_barrier_wait(&start_barrier)` releases all 5 threads at the same time, so they run the same steps together.
- Each thread locks its **left** fork first: `pthread_mutex_lock(&forks[left]);`  
    After this, fork `i` is held by philosopher `i`.
- `usleep(200ms)` gives enough time for **all** threads to successfully lock their left fork before anyone tries to lock the right fork.
- Then every thread tries to lock its **right** fork: `pthread_mutex_lock(&forks[right]);`

which causes deadlock.


```bash
[beyza@suvari ~]$ sudo gdb -p 15183

For help, type "help".
Type "apropos word" to search for commands related to "word".
Attaching to process 15183
[New LWP 15188]
[New LWP 15187]
[New LWP 15186]
[New LWP 15185]
[New LWP 15184]
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/usr/lib/libthread_db.so.1".
0x00007fa705b6cf32 in ?? () from /usr/lib/libc.so.6
(gdb) thread apply all bt

Thread 6 (Thread 0x7fa705aca6c0 (LWP 15184) "test"):
#0  0x00007fa705b618d0 in ?? () from /usr/lib/libc.so.6
#1  0x00007fa705b680c4 in pthread_mutex_lock () from /usr/lib/libc.so.6
#2  0x000055e65b4c22de in philosopher ()
#3  0x00007fa705b6497a in ?? () from /usr/lib/libc.so.6
#4  0x00007fa705be82bc in ?? () from /usr/lib/libc.so.6

Thread 5 (Thread 0x7fa7052c96c0 (LWP 15185) "test"):
#0  0x00007fa705b618d0 in ?? () from /usr/lib/libc.so.6
#1  0x00007fa705b680c4 in pthread_mutex_lock () from /usr/lib/libc.so.6
#2  0x000055e65b4c22de in philosopher ()
#3  0x00007fa705b6497a in ?? () from /usr/lib/libc.so.6
#4  0x00007fa705be82bc in ?? () from /usr/lib/libc.so.6

Thread 4 (Thread 0x7fa704ac86c0 (LWP 15186) "test"):
#0  0x00007fa705b618d0 in ?? () from /usr/lib/libc.so.6
#1  0x00007fa705b680c4 in pthread_mutex_lock () from /usr/lib/libc.so.6
#2  0x000055e65b4c22de in philosopher ()
#3  0x00007fa705b6497a in ?? () from /usr/lib/libc.so.6
#4  0x00007fa705be82bc in ?? () from /usr/lib/libc.so.6

Thread 3 (Thread 0x7fa7042c76c0 (LWP 15187) "test"):
#0  0x00007fa705b618d0 in ?? () from /usr/lib/libc.so.6
#1  0x00007fa705b680c4 in pthread_mutex_lock () from /usr/lib/libc.so.6
#2  0x000055e65b4c22de in philosopher ()
#3  0x00007fa705b6497a in ?? () from /usr/lib/libc.so.6
#4  0x00007fa705be82bc in ?? () from /usr/lib/libc.so.6
--Type <RET> for more, q to quit, c to continue without paging--c

Thread 2 (Thread 0x7fa703ac66c0 (LWP 15188) "test"):
#0  0x00007fa705b618d0 in ?? () from /usr/lib/libc.so.6
#1  0x00007fa705b680c4 in pthread_mutex_lock () from /usr/lib/libc.so.6
#2  0x000055e65b4c22de in philosopher ()
#3  0x00007fa705b6497a in ?? () from /usr/lib/libc.so.6
#4  0x00007fa705be82bc in ?? () from /usr/lib/libc.so.6

Thread 1 (Thread 0x7fa705acb740 (LWP 15183) "test"):
#0  0x00007fa705b6cf32 in ?? () from /usr/lib/libc.so.6
#1  0x00007fa705b6139c in ?? () from /usr/lib/libc.so.6
#2  0x00007fa705b6168c in ?? () from /usr/lib/libc.so.6
#3  0x00007fa705b667b5 in ?? () from /usr/lib/libc.so.6
#4  0x000055e65b4c2431 in main ()
```

what do we see from the output:

there are 6 threads, one of them is main (with LWP (pid for threads) 15183)
```
Attaching to process 15183
[New LWP 15188]
[New LWP 15187]
[New LWP 15186]
[New LWP 15185]
[New LWP 15184]
```

stack frame numbers: a backtrace shows the chain of function calls that led to the current point.
- `#0` = the **current function** where the thread is stopped _right now_ (top of the stack)
- `#1` = the **caller** of `#0`
- `#2` = the caller of `#1`
- ...and so on, going "back in time" through the call chain

Example:
```
#0  ... in ?? () from /usr/lib/libc.so.6
#1  ... in pthread_mutex_lock () from /usr/lib/libc.so.6
#2  ... in philosopher ()
#3  ... in ?? () from /usr/lib/libc.so.6
```
- The thread is currently executing some internal libc wait routine (`#0`, shown as `??`)
- That routine was called by `pthread_mutex_lock` (`#1`)
- `pthread_mutex_lock` was called by your function `philosopher()` (`#2`)
- `philosopher()` was called by the thread start wrapper inside libc (`#3`)

So the code called `pthread_mutex_lock()`, and the thread is stuck inside it.

from the output we can infer:

- All worker threads are blocked in `pthread_mutex_lock()` (none of them are executing code that would `pthread_mutex_unlock()`).
- So no one will release any fork.
- The main thread is blocked waiting in `main()`.

This combination is a typical "program is deadlocked / stuck" signature.

Because the backtrace includes:

`#2 0x... in philosopher ()`

we know the hang is happening **inside the philosopher function**, not in thread creation, not in I/O, not somewhere else.

how do we know which thread wants to lock which one? 

--i've restarted the program here so ignore the changed LWPs

because `$rdi` is the first argument to `pthread_mutex_lock(pthread_mutex_t *mutex)` on **x86_64 SysV ABI**, we can print the **address of the mutex that this specific thread is currently trying to lock**.

```
(gdb) thread 2
[Switching to thread 2 (Thread 0x7f5860c026c0 (LWP 17800))]
#0  0x00007f5862c9d8d0 in ?? () from /usr/lib/libc.so.6
(gdb) frame 1
#1  0x00007f5862ca40c4 in pthread_mutex_lock () from /usr/lib/libc.so.6
(gdb) p/x $rdi
$1 = 0x55a2d08c70a0
(gdb) thread 3
[Switching to thread 3 (Thread 0x7f58614036c0 (LWP 17799))]
#0  0x00007f5862c9d8d0 in ?? () from /usr/lib/libc.so.6
(gdb) frame 1
#1  0x00007f5862ca40c4 in pthread_mutex_lock () from /usr/lib/libc.so.6
(gdb) p/x $rdi
$2 = 0x55a2d08c7140
(gdb) thread 4
[Switching to thread 4 (Thread 0x7f5861c046c0 (LWP 17798))]
#0  0x00007f5862c9d8d0 in ?? () from /usr/lib/libc.so.6
(gdb) frame 1
#1  0x00007f5862ca40c4 in pthread_mutex_lock () from /usr/lib/libc.so.6
(gdb) p/x $rdi
$3 = 0x55a2d08c7118
(gdb) thread 5
[Switching to thread 5 (Thread 0x7f58624056c0 (LWP 17797))]
#0  0x00007f5862c9d8d0 in ?? () from /usr/lib/libc.so.6
(gdb) frame 1
#1  0x00007f5862ca40c4 in pthread_mutex_lock () from /usr/lib/libc.so.6
(gdb) p/x $rdi
$4 = 0x55a2d08c70f0
(gdb) thread 6
[Switching to thread 6 (Thread 0x7f5862c066c0 (LWP 17796))]
#0  0x00007f5862c9d8d0 in ?? () from /usr/lib/libc.so.6
(gdb) frame 1
#1  0x00007f5862ca40c4 in pthread_mutex_lock () from /usr/lib/libc.so.6
(gdb) p/x $rdi
$5 = 0x55a2d08c70c8
```

When a thread is blocked here:

```c
pthread_mutex_lock(&forks[right]);
```
it is literally calling the function:

```c
pthread_mutex_lock(pthread_mutex_t *mutex);
```

So at the moment it's inside `pthread_mutex_lock`, the thread must "remember" which mutex pointer it was asked to lock. That pointer is an argument to the function call.

Your trick in `gdb` was: **read that argument from the CPU register where the ABI puts it**.

so, `$rdi` is not "the lock"
It's the **address of the mutex object** (the thing being locked).


Because  `forks` are stored in a contiguous array:

```c
pthread_mutex_t forks[5];
```

Memory looks like:

- `&forks[0]` at some base address, say `B`
- `&forks[1]` at `B + sizeof(pthread_mutex_t)`
- `&forks[2]` at `B + 2*sizeof(pthread_mutex_t)`
- ...

So if a thread's `$rdi` equals `B + 3*sizeof(pthread_mutex_t)`, then it's trying to lock `forks[3]`.

thats why i computed offset from the array start:

```c
(gdb) p/x (unsigned long)$rdi - (unsigned long)&forks
$14 = 0x28
```

and divide by element size (`0x28` on my system):

to see which thread got which fork:

```c
(gdb) thread 2
[Switching to thread 2 (Thread 0x7f5860c026c0 (LWP 17800))]
#0  0x00007f5862c9d8d0 in ?? () from /usr/lib/libc.so.6
(gdb) p ((unsigned long)$rdi - (unsigned long)&forks) / 0x28
$9 = 0
(gdb) thread 3
[Switching to thread 3 (Thread 0x7f58614036c0 (LWP 17799))]
#0  0x00007f5862c9d8d0 in ?? () from /usr/lib/libc.so.6
(gdb) p ((unsigned long)$rdi - (unsigned long)&forks) / 0x28
$10 = 4
(gdb) thread 4
[Switching to thread 4 (Thread 0x7f5861c046c0 (LWP 17798))]
#0  0x00007f5862c9d8d0 in ?? () from /usr/lib/libc.so.6
(gdb) p ((unsigned long)$rdi - (unsigned long)&forks) / 0x28
$11 = 3
(gdb) thread 5
[Switching to thread 5 (Thread 0x7f58624056c0 (LWP 17797))]
#0  0x00007f5862c9d8d0 in ?? () from /usr/lib/libc.so.6
(gdb) p ((unsigned long)$rdi - (unsigned long)&forks) / 0x28
$12 = 2
(gdb) thread 6
[Switching to thread 6 (Thread 0x7f5862c066c0 (LWP 17796))]
#0  0x00007f5862c9d8d0 in ?? () from /usr/lib/libc.so.6
(gdb) p ((unsigned long)$rdi - (unsigned long)&forks) / 0x28
$13 = 1
```



Real-world deadlocks often involve complex lock dependencies across subsystems. A database transaction may hold row locks while waiting for a log buffer lock, while another thread holds the log buffer and waits for row locks. Detecting these requires:

- Logging all lock acquisitions and releases with timestamps and thread IDs
- Building the resource allocation graph from this log
- Running cycle detection algorithms on the graph

A simplified approach uses timeout-based detection. Most pthread mutexes support the `PTHREAD_MUTEX_ERRORCHECK` type or can be used with `pthread_mutex_timedlock`:

```c
pthread_mutexattr_t attr;
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_ERRORCHECK);
pthread_mutex_init(&mutex, &attr);

// Later, pthread_mutex_lock will return EDEADLK if called twice
// by same thread, helping detect self-deadlock
```

Consistent lock ordering is the fundamental deadlock prevention strategy. If all threads acquire locks in the same global order, circular wait cannot form. This requires documentation and enforcement:

```c
// Lock ordering rule: always acquire locks in order database -> cache -> log
// Never acquire in reverse order or hold multiple locks in undefined order

void transfer_funds(int from_id, int to_id, int amount) {
    // Ensure consistent ordering by ID
    int first = (from_id < to_id) ? from_id : to_id;
    int second = (from_id < to_id) ? to_id : from_id;
    
    pthread_mutex_lock(&account_locks[first]);
    pthread_mutex_lock(&account_locks[second]);
    // perform transfer
    pthread_mutex_unlock(&account_locks[second]);
    pthread_mutex_unlock(&account_locks[first]);
}
```






## Tools for Thread Debugging (Valgrind/Helgrind, ThreadSanitizer)

Dynamic analysis tools provide systematic detection of concurrency bugs by instrumenting the running program and tracking memory accesses and synchronization operations. Unlike static analysis, dynamic analysis observes actual execution paths but may miss bugs that only manifest under specific interleavings not exercised during the monitored run.

**Helgrind** is a Valgrind-based tool specialized for detecting synchronization errors in programs using the pthreads library. It implements a happens-before analysis at runtime, tracking lock acquisitions and releases, thread creation and joins, and condition variable operations. Helgrind builds a precise graph of inter-thread ordering edges and reports when memory accesses lack sufficient ordering constraints.

When Helgrind runs a program, it maintains a vector clock for each thread, which captures the logical time at each synchronization point. A lock acquisition updates the acquiring thread's clock with the releasing thread's clock information, establishing happens-before. Memory writes record the thread's clock at that point, and subsequent reads check whether the read is ordered after the write.

Running a program under Helgrind:

```bash
valgrind --tool=helgrind ./my_program arg1 arg2
```

Helgrind's output identifies potential race conditions with precise information:

```
 Possible data race during write of size 4 at 0x404060 by thread #3
 Locks held: none
    at 0x4011A0: thread_func (example.c:15)
    by 0x4E3F6F4: start_thread (pthread_create.c:309)
 
 This conflicts with a previous read of size 4 by thread #2
 Locks held: 1, at 0x404080
    at 0x4011C5: other_func (example.c:22)
```

The output shows the conflicting memory address, the thread performing each conflicting access, and the locks held (or not held) during each access. This immediately reveals which variables lack proper synchronization.

Helgrind also detects lock order violations. When it observes locks being acquired in inconsistent orders across different execution paths, it reports potential deadlock scenarios even if no deadlock has occurred:

```
 Thread #1: lock order "0x404080 before 0x4040A0" established
 ...
 Thread #2: lock order "0x4040A0 before 0x404080" violated
```

**ThreadSanitizer (TSan)** is a data race detector built into GCC and Clang that provides similar functionality with lower overhead than Valgrind-based tools. It instruments the compiled binary to track memory accesses and synchronization events, using a compact representation that allows it to handle programs with millions of synchronization events.

Compiling with ThreadSanitizer:

```bash
gcc -fsanitize=thread -g -O1 -pthread example.c -o example
clang -fsanitize=thread -g -O1 -pthread example.c -o example
```


here is a quick example of data race:

```c
[beyza@suvari leetcodee]$ gcc -o test test.c
[beyza@suvari leetcodee]$ ./test
thread 0 increased a
a = 3
thread 1 increased a
a = 3
thread 3 increased a
a = 4
thread 2 increased a
a = 4
thread 4 increased a
a = 5



[beyza@suvari leetcodee]$ gcc -fsanitize=thread -g -O1 -pthread test.c -o test
[beyza@suvari leetcodee]$ ./test
thread 0 increased a
a = 1
thread 2 increased a
a = 2
==================
WARNING: ThreadSanitizer: data race (pid=22190)
  Read of size 4 at 0x55db1439d168 by thread T2:
    #0 philosopher /home/beyza/obsidian/vscode/leetcodee/test.c:19 (test+0x11f3) (BuildId: 624e7c5cb1464e1a8fd22d49e8596df4acd315d7)
    #1 <null> <null> (libtsan.so.2+0x541b9) (BuildId: f29521f558650bcc384c0178d8c6d0fd49466e29)

  Previous write of size 4 at 0x55db1439d168 by thread T1:
    #0 philosopher /home/beyza/obsidian/vscode/leetcodee/test.c:19 (test+0x1204) (BuildId: 624e7c5cb1464e1a8fd22d49e8596df4acd315d7)
    #1 <null> <null> (libtsan.so.2+0x541b9) (BuildId: f29521f558650bcc384c0178d8c6d0fd49466e29)

  Location is global 'a' of size 4 at 0x55db1439d168 (test+0x4168)

  Thread T2 (tid=22193, running) created by main thread at:
    #0 pthread_create <null> (libtsan.so.2+0x5fb47) (BuildId: f29521f558650bcc384c0178d8c6d0fd49466e29)
    #1 main /home/beyza/obsidian/vscode/leetcodee/test.c:32 (test+0x12a4) (BuildId: 624e7c5cb1464e1a8fd22d49e8596df4acd315d7)

  Thread T1 (tid=22192, finished) created by main thread at:
    #0 pthread_create <null> (libtsan.so.2+0x5fb47) (BuildId: f29521f558650bcc384c0178d8c6d0fd49466e29)
    #1 main /home/beyza/obsidian/vscode/leetcodee/test.c:32 (test+0x12a4) (BuildId: 624e7c5cb1464e1a8fd22d49e8596df4acd315d7)

SUMMARY: ThreadSanitizer: data race /home/beyza/obsidian/vscode/leetcodee/test.c:19 in philosopher
==================
==================
WARNING: ThreadSanitizer: data race (pid=22190)
  Write of size 4 at 0x55db1439d168 by thread T2:
    #0 philosopher /home/beyza/obsidian/vscode/leetcodee/test.c:19 (test+0x1204) (BuildId: 624e7c5cb1464e1a8fd22d49e8596df4acd315d7)
    #1 <null> <null> (libtsan.so.2+0x541b9) (BuildId: f29521f558650bcc384c0178d8c6d0fd49466e29)

  Previous write of size 4 at 0x55db1439d168 by thread T3:
    #0 philosopher /home/beyza/obsidian/vscode/leetcodee/test.c:19 (test+0x1204) (BuildId: 624e7c5cb1464e1a8fd22d49e8596df4acd315d7)
    #1 <null> <null> (libtsan.so.2+0x541b9) (BuildId: f29521f558650bcc384c0178d8c6d0fd49466e29)

  Location is global 'a' of size 4 at 0x55db1439d168 (test+0x4168)

  Thread T2 (tid=22193, running) created by main thread at:
    #0 pthread_create <null> (libtsan.so.2+0x5fb47) (BuildId: f29521f558650bcc384c0178d8c6d0fd49466e29)
    #1 main /home/beyza/obsidian/vscode/leetcodee/test.c:32 (test+0x12a4) (BuildId: 624e7c5cb1464e1a8fd22d49e8596df4acd315d7)

  Thread T3 (tid=22194, finished) created by main thread at:
    #0 pthread_create <null> (libtsan.so.2+0x5fb47) (BuildId: f29521f558650bcc384c0178d8c6d0fd49466e29)
    #1 main /home/beyza/obsidian/vscode/leetcodee/test.c:32 (test+0x12a4) (BuildId: 624e7c5cb1464e1a8fd22d49e8596df4acd315d7)

SUMMARY: ThreadSanitizer: data race /home/beyza/obsidian/vscode/leetcodee/test.c:19 in philosopher
==================
thread 1 increased a
a = 3
thread 3 increased a
a = 4
thread 4 increased a
a = 5
ThreadSanitizer: reported 2 warnings
```


The report identifies the variable name (when symbols are available), the location in source code for both accesses, and distinguishes read from write operations.


Key conditions for effective use of dynamic race detectors:

- The program must actually execute the code paths containing the race condition
- Sufficient test coverage with varied inputs is essential; races not exercised cannot be detected
- Long-running programs may need extended testing time for races to manifest
- Thread interleavings depend on scheduler behavior, which varies across systems and load conditions

Neither Helgrind nor ThreadSanitizer guarantees finding all race conditions, they report races observed during execution. This means extensive test suites are crucial, and false positives (races that are intentional or benign) must be distinguished from actual bugs. False positives can be suppressed through annotation:



```c
// For ThreadSanitizer
void function_with_known_race(void) {
    TSAN_ANNOTATE_HAPPENS_BEFORE(&flag);
    // code with intentional unsynchronized access
    TSAN_ANNOTATE_HAPPENS_AFTER(&flag);
}

// For Helgrind (using VALGRIND macros)
#include <valgrind/helgrind.h>
VALGRIND_HG_DISABLE_CHECKING(&variable, sizeof(variable));
```

Both tools also detect misuse of pthreads APIs, such as:

- Unlocking a mutex owned by a different thread
- Attempting to lock a non-recursive mutex twice from the same thread
- Destroying a locked mutex
- Waiting on a condition variable without holding its associated mutex
- Failing to recheck the condition after `pthread_cond_wait` returns

These API violations often indicate logic errors that could lead to undefined behavior or extremely subtle bugs that manifest only under specific timing. The tools identify these at the point of violation, significantly reducing debugging time.
