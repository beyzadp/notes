---
title: "3. Data Races and Critical Sections"
---

part of the [[index|posix-threads]] series.

In concurrent programming, multiple threads or processes often need to access and modify shared memory. If these accesses are not carefully managed, unpredictable bugs, called **data races**, can occur. Data races happen when two or more threads access the same memory location concurrently, and at least one of the accesses is a write. This unpredictability leads to inconsistent program behavior, hard-to-trace bugs, and broken system invariants.

If critical zones of memory (or shared data) are left unprotected, legacy and new systems alike risk silent corruption or crashes under increased concurrency.



## Defining the Race Condition

A **race condition** occurs when the outcome of a program depends on the timing or interleaving of threads, rather than just the program logic. When two threads read-modify-write the same variable without coordination, their operations can overlap in unpredictable ways.

**Key indicators of a race condition:**
- The program produces different results on different runs.
- Bugs disappear or change behavior when adding print statements or delays.
- Issues surface only under real customer load, never in single-threaded test runs.

Remember when i said multithreaded programs exhibit non-deterministic execution (different thread interleavings produce different results), making bugs sporadic and hard to reproduce. This is one of the mentioned bugs.


If two or more actors can operate on the same object or value without an explicit order or coordination, expect a race condition unless it is designed out or handled.


## Atomicity and Interleaving

When several threads are running, it's easy to **think** of operations (like incrementing a variable or updating a status) as happening "all at once." In reality, most operations are **not** indivisible, they are made up of multiple lower-level steps. Problems arise when two (or more) threads overlap or "interleave" these steps, producing incorrect or unpredictable results:

- **Atomicity** ensures actions happen in one, indivisible block, either fully completed or not seen at all by any other thread. it cannot be observed in an unfinished state by other threads.

- **Interleaving** describes what actually happens: the underlying processor and operating system split each thread's work into small pieces, mixing steps from different threads unpredictably for performance. The CPU can switch between threads at any time and can "slice up" each thread's instructions, executing pieces in an order not visible (or controllable) by your code.

an example of atomic operation:
```c
#include <stdatomic.h>
atomic_int counter = 0;

// In each thread:
atomic_fetch_add(&counter, 1); // Atomic increment
```

`atomic_fetch_add` ensures a **single, indivisible operation**: each increment will be seen by all threads with no steps "between". which means no other thread can read the counter in a half-updated state.

and here is an example of interleaved operation:

```c
int counter = 0;

// In each thread:
counter = counter + 1; // Not atomic!
```

it seems like it's just incremented at once, but here is what actually happens:
1. Thread A reads `counter` 
2. Thread A adds 1
3. Thread A writes the result to `counter`

**What might actually happen with multithreading (compiled/CPU steps):**

1. Thread A reads `counter` (e.g., gets 0)
2. Thread B reads `counter` (also 0)
3. Thread A adds 1 (result: 1)
4. Thread B adds 1 (result: 1)
5. Thread A writes 1 to `counter`
6. Thread B writes 1 to `counter`

The counter is incremented only once, even though both threads "believe" they incremented it.

so:
Simple assignments, pointer swaps, and field writes are not guaranteed atomic except for certain machine types and data sizes. even if the code looks safe instruction reordering (compiler or CPU) and visibility issues (caches) can break guarantees.

## The Critical Section Problem


A **critical section** is a region of code where shared mutable resources are accessed and must not be executed concurrently by more than one thread or process. All state changes to shared resources *must* happen entirely within a critical section protected by some synchronization mechanism.


### Characteristics of Critical Sections:

- They encapsulate potentially dangerous state transitions in a well-defined "protected zone".
- Any thread entering a critical section must first acquire the relevant lock or synchronization guard.
- On exit, the lock/guard is released, allowing others to enter.
- The set of variables or resources a critical section protects must be clearly documented; misunderstanding scope leads to subtle bugs.

### Recognizing a Critical Section:
- Operations that:
    - Update counters, balances, shared pointers, or resource trackers.
    - Make changes that depend on the current value (e.g., "if available, allocate; else, fail").
    - Maintain global or cross-thread invariants.
    - Build or tear down parts of larger data structures (e.g., hash tables, linked lists).


## Invariants and Shared State


**Invariants** are properties or logical relationships between data that must *always* hold true, regardless of the sequence or concurrency of operations. When shared state is updated unsafely, these invariants can be destroyed.

for example if your code has this condition in it:

```c
if (counter > 0) {
	counter = counter - 1;
}
```

counter should never be negative. it can be possible only with race conditions. if it has a negative number, this breaks the invariant.

**Why invariants matter:**
- They simplify reasoning about correctness: You know what must be true, even if you don't know exactly what has happened. 

Shared state refers to variables or data that can be accessed and modified by multiple threads at the same time.

We care about shared state because it threatens our invariants.  
Keeping data safe from race conditions **means protecting shared state to ensure invariants always hold**.












## Concrete Example Scenario

here i have two counters, one of them is protected and the other one is not:

```c
//unprotected_counter.c
// this is an unprotected code for race conditions

#include <pthread.h>
#include <stdio.h>

int counter = 0; // Shared variable

void *increment(void *arg) {
    for (int i = 0; i < 100000; i++) {
        counter = counter + 1; // Not protected: Race condition possible
    }
    return NULL;
}

int main() {
    pthread_t t1, t2;

    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("Final counter: %d\n", counter); // Often less than 200000
    return 0;
}
```



```c
//protected_counter.c
#include <pthread.h>
#include <stdio.h>

int counter = 0; // Shared variable
pthread_mutex_t counter_mutex = PTHREAD_MUTEX_INITIALIZER;

void *increment(void *arg) {
    for (int i = 0; i < 100000; i++) {
        pthread_mutex_lock(&counter_mutex);   // Begin critical section
        counter = counter + 1;                // Safe update
        pthread_mutex_unlock(&counter_mutex); // End critical section
    }
    return NULL;
}

int main() {
    pthread_t t1, t2;

    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("Final counter: %d\n", counter); // Always 200000
    return 0;
}
```




as you can see below unprotected one returns different values each time. but safe code returns the right value.

```bash
[beyza@suvari leetcodee]$ gcc -pthread unprotected_counter.c -o unprotected_counter
[beyza@suvari leetcodee]$ ./unprotected_counter 
Final counter: 100000
[beyza@suvari leetcodee]$ ./unprotected_counter 
Final counter: 151562
[beyza@suvari leetcodee]$ ./unprotected_counter 
Final counter: 138824
[beyza@suvari leetcodee]$ ./unprotected_counter 
Final counter: 126824
[beyza@suvari leetcodee]$ ./unprotected_counter 
Final counter: 127712
[beyza@suvari leetcodee]$ gcc -pthread protected_counter.c -o protected_counter
[beyza@suvari leetcodee]$ ./protected_counter 
Final counter: 200000
[beyza@suvari leetcodee]$ ./protected_counter 
Final counter: 200000
[beyza@suvari leetcodee]$ ./protected_counter 
Final counter: 200000
[beyza@suvari leetcodee]$ ./protected_counter 
Final counter: 200000
```


```bash
[beyza@suvari leetcodee]$ hyperfine ./unprotected_counter -r 1000
Benchmark 1: ./unprotected_counter
  Time (mean ± σ):       2.2 ms ±   0.6 ms    [User: 3.0 ms, System: 0.8 ms]
  Range (min … max):     1.0 ms …   4.7 ms    1000 runs
 

 
[beyza@suvari leetcodee]$ hyperfine ./protected_counter -r 1000
Benchmark 1: ./protected_counter
  Time (mean ± σ):      10.9 ms ±   1.7 ms    [User: 8.9 ms, System: 10.9 ms]
  Range (min … max):     5.2 ms …  16.1 ms    1000 runs
 
```

and it takes almost 5x the time to run the protected code.

but if each thread in the program maintained its own private counter and incremented it independently, avoiding any shared memory updates during execution. Once both threads finished, the main function summed the two counters to get the total. This approach prevents race conditions and synchronization overhead because threads do not compete for the same variable; it's a classic example of thread-local storage (see [[08-thread-local-storage]]) and result reduction in parallel programming.

so there are ways to do this correctly: the mutex-based fix is in [[04-mutexes]], right next.
