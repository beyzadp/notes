---
title: "7. Advanced Synchronization Primitives"
---

part of the [[index|posix-threads]] series.

## Read-Write Locks (`pthread_rwlock_t`)

A `pthread_rwlock_t` splits access into two modes: **shared** (read) and **exclusive** (write). The lock maintains internal state tracking the number of active readers, whether an active writer exists, and which waiters are queued. The key optimization is that read-side critical sections can proceed concurrently when there is no writer holding (and depending on policy, sometimes no writer waiting).

Internally, read-write locks may be implemented with an internal mutex plus condition variables, or with futex-based state machines on Linux. 

readers can scale well under read-heavy workloads; multiple readers can enter concurrently. but writers must wait for all readers to drain because a writer needs exclusive access. If there are currently `N` readers inside the critical section, the writer can't proceed until:

- each of those `N` readers calls `pthread_rwlock_unlock`, and
- the internal `reader_count` drops to zero

Even if the writer arrived first, it still cannot "push out" current readers; it must wait for them to finish naturally. If readers hold the lock for a long time, write latency becomes large, and wake-up policy can cause bursts of readers to reacquire repeatedly. here is what i mean:

1. Many readers are running.
2. A writer arrives and starts waiting.
3. Readers finish and unlock around the same time.
4. The lock becomes "available".

Now the implementation must choose who to wake/admit:

- If it wakes **all waiting readers** (or allows new readers to barge in), you can get a **reader burst**: lots of readers immediately reacquire the lock.
- That reacquisition increments `reader_count` again, so the writer still cannot proceed.
- If this keeps happening (constant stream of incoming readers), the writer can be delayed for a very long time (**writer starvation**).

Even if the writer is awakened, there's often a race: after the last reader unlocks, newly scheduled readers may run first and grab the read lock before the writer successfully transitions the state to "writer active". That "reacquire repeatedly" behavior is essentially _readers winning the race repeatedly_.



### Read Locks vs. Write Locks


A **read lock** (`pthread_rwlock_rdlock`) grants shared access: multiple threads may hold it simultaneously as long as no writer holds the lock. A **write lock** (`pthread_rwlock_wrlock`) grants exclusive access: exactly one thread holds it, and it excludes both other writers and all readers. a **read lock** (`pthread_rwlock_rdlock`) only makes the code correct if _every thread that holds a read lock truly performs only reads._ even "benign" mutations like lazy initialization can violate invariants unless carefully designed (e.g., double-checked locking with atomics).

heres an incorrect pattern:

```c
// shared
static pthread_rwlock_t rw = PTHREAD_RWLOCK_INITIALIZER;
static int initialized = 0;
static int expensive_value;

int get_value(void) {
    pthread_rwlock_rdlock(&rw);

    if (!initialized) {
        // BUG: writing while holding only a read lock
        expensive_value = 42;
        initialized = 1;
    }

    int v = expensive_value;
    pthread_rwlock_unlock(&rw);
    return v;
}
```

many threads can hold the read lock at the same time:

- Two threads can both see `initialized == 0` and both write initialization concurrently.
- Worse, another thread might observe `initialized == 1` **before** `expensive_value` is fully written (or before all related fields are consistent), depending on reordering and visibility.
- If initialization involves multiple fields (struct with pointers, sizes, etc.), readers can see a **half-built object**.

This breaks the assumptions that made it safe to let readers run concurrently.

From a memory-ordering standpoint, `pthread_rwlock_*lock` operations act as acquire barriers and unlock operations act as release barriers in POSIX threads practice: writes in a critical section become visible to threads that subsequently acquire the lock. Practically, this means you can use the lock to publish data safely without additional atomics, but you must ensure all accesses follow the lock discipline.

A frequent performance subtlety is that read locks are not "free": they still update shared state (reader counts) and can contend on cache lines. Under extremely high thread counts, reader-count updates can become a bottleneck, making an `rwlock` slower than a plain `mutex` despite more concurrency.





### RWLock safety

Use `pthread_rwlock_t` only if you follow this discipline:

- **`rdlock` = read-only.** While holding a read lock, do **not** modify *anything* in the protected state (no flags, no lazy init, no counters, no pointer updates).
- **`wrlock` = all writes.** Any mutation of protected data must be done while holding the **write lock**, and the whole multi-step update must stay inside one write-locked critical section.
- **No unlocked access.** Every read/write of the protected data must happen under the lock.
- **Don't keep pointers after unlock.** Anything you take a pointer/reference to is only safe to use while the lock is still held (unless you have separate lifetime management).
- **No lock upgrade.** Never call `wrlock` while holding `rdlock` on the same lock; release and reacquire with a design that tolerates the race window, or redesign.

If you violate any of the above, you can get **data race**, **broken invariants**, or **writer starvation/liveness issues**.


**Typical usage pattern (C/pthreads):**
```c
#include <pthread.h>
#include <stdio.h>

static pthread_rwlock_t rw = PTHREAD_RWLOCK_INITIALIZER;
static int shared_value = 0;

int read_value(void) {
    pthread_rwlock_rdlock(&rw);
    int v = shared_value;          // safe concurrent read
    pthread_rwlock_unlock(&rw);
    return v;
}

void write_value(int v) {
    pthread_rwlock_wrlock(&rw);
    shared_value = v;              // exclusive update
    pthread_rwlock_unlock(&rw);
}
```

This works because all reads and writes of `shared_value` are covered by the lock. Readers can run concurrently, but writes serialize and exclude readers.




### Starvation Issues


Starvation arises when scheduling and lock admission rules allow one class of waiter to be postponed indefinitely. ive mentioned about the starvation issues a few section earlier. With `pthread_rwlock_t`, the classic case is **writer starvation**: a constant stream of incoming readers repeatedly acquires the read lock, preventing a waiting writer from ever seeing the reader count drop to zero.

The root cause is policy. Many `rwlock` implementations allow readers to acquire the lock when no writer *holds* it, even if a writer is *waiting*. If new readers are admitted while a writer is queued, the writer's condition ("no readers active") may never become true. Even if readers eventually drain, a thundering herd of readers waking and re-acquiring can repeatedly beat the writer in the race. 

Mitigations depend on what you can control:
- If the implementation has a writer-preference policy, enabling it can reduce starvation. On glibc, `pthread_rwlockattr_setkind_np(&attr, PTHREAD_RWLOCK_PREFER_WRITER_NONRECURSIVE_NP)` sets this before creating the lock.
- At the design level, reduce read-lock hold time and avoid long reader critical sections.
- In some designs, a "phase-fair" approach is used: once a writer arrives, new readers are blocked until the writer proceeds, preventing indefinite postponement.
- there are also different primitives you can use. like `pthread_mutex_t` or a custom `rwlock` with explicit writer preference.

### Designs

- **Problem targeted:** reader-preference policies can cause **writer starvation** under steady read traffic.
- **Designs to prevent/mitigate starvation (choose by goal):**
    - **Writer-preference `rwlock` (custom or platform-specific)**
        - **Use when:** you still want concurrent reads, but **writers must not starve** (write latency matters).
        - **Design idea:** once a writer is waiting, **block new readers** until queued writers run (often called _phase-fair_ / _writer-gated_).
        - **Tradeoff:** reduced read throughput during write pressure; higher predictability.
    - **Phase-fair RW lock (explicit "read phase" / "write phase")**
        - **Use when:** you want a stronger fairness story than "writer preference", especially under high churn.
        - **Design idea:** alternate phases; if a writer arrives, let current readers drain, then run writer(s), then reopen reads.
    - **Switch to `pthread_mutex_t` instead of `rwlock`**
        - **Use when:** fairness + simplicity > maximum read parallelism, or when the workload is not truly read-heavy.
        - **Why it helps:** a mutex often has **better practical fairness** than many `rwlock` reader-biased implementations, and removes the "new readers keep sneaking in" pattern.
        - **Tradeoff:** no concurrent readers.
    - **Bound read-side critical sections (design rule)**
        - **Use when:** you must keep `rwlock` but can control code structure.
        - **Design idea:** ensure reads don't do I/O/page faults/long work while holding `rdlock`; move long-latency work outside lock.
        - **Effect:** reduces worst-case writer wait, lowering starvation risk.



## Spinlocks (`pthread_spinlock_t`)

A `pthread_spinlock_t` is a lock that provides **mutual exclusion** (only one thread can own it at a time), like a `pthread_mutex_t`. The big difference is what happens when the lock is already held:

- With a **mutex**, the waiting thread typically **sleeps** (blocks). The OS can deschedule it, and later wake it when the lock becomes available.
- With a **spinlock**, the waiting thread does **busy-waiting**: it repeatedly checks/tries to take the lock in a tight loop, consuming CPU cycles until it succeeds.

This is only a good trade when the expected wait is extremely short, shorter than the overhead of putting a thread to sleep and waking it up. They are most appropriate for *very short* critical sections on multi-core systems where the lock holder is likely running concurrently and will release soon. If the holder is descheduled, spinning can waste an entire CPU core doing no useful work.

**API behaviors / usage rules:**
- `pthread_spin_init(pthread_spinlock_t *lock, int pshared)` initializes; `pshared` controls process-sharing in some environments.
- `pthread_spin_lock` spins until acquired.
- `pthread_spin_trylock` returns immediately if not available.
- `pthread_spin_unlock` releases.

**Example: Using a spinlock to protect a pointer swap**

```c
#include <pthread.h>
#include <stdlib.h>

static pthread_spinlock_t slock;

struct config {
    int a, b;
};

static struct config *g_cfg;

void update_config(int a, int b) {
    struct config *new_cfg = malloc(sizeof(*new_cfg));
    new_cfg->a = a;
    new_cfg->b = b;

    pthread_spin_lock(&slock);
    struct config *old = g_cfg;
    g_cfg = new_cfg;                 // tiny critical section: just pointer ops
    pthread_spin_unlock(&slock);

    free(old);
}

struct config snapshot_config(void) {
    pthread_spin_lock(&slock);
    struct config copy = *g_cfg;     // copy while protected
    pthread_spin_unlock(&slock);
    return copy;
}
```


**Common mistakes / pitfalls:**
- Spinning around code that can block (I/O, `malloc`, page faults, syscalls). A spinlock must protect only code that is guaranteed to be short and non-blocking.
- Using spinlocks on a single-core system: progress can collapse because the spinner prevents the lock holder from running (unless the OS time-slices, which still wastes cycles).
- Ignoring fairness: spinlocks often have weak fairness; a thread can repeatedly lose the race and starve under contention.

**Safe to use a spinlock for (short + non-blocking)**

- Updating a few shared counters/flags
- Swapping pointers (publish/replace a pointer value)
- Updating indices/pointers in a fixed-size ring buffer
- Pushing/popping on a small lock-protected freelist (no allocation)
- Reading/updating a small struct where all fields must change together
- Protecting a very short critical section in a hot path where the lock is rarely contended

**Not safe to use a spinlock for (can block or take "long/unbounded" time)**

- `malloc` / `free` (can take locks internally, can be slow/unbounded)
- Any I/O (`read`, `write`, `printf`, file/network operations)
- Sleeping/yielding (`sleep`, `usleep`, `nanosleep`)
- Waiting primitives (`pthread_cond_wait`, `sem_wait`, `pthread_join`)
- System calls that may block (many do, depending on state)
- Taking another contended lock (risk of deadlock + long wait)
- Page-fault-prone work (touching lots of new memory, mapping files)
- Long loops / heavy computation / CPU-heavy work


## Barriers (`pthread_barrier_t`)


A `pthread_barrier_t` provides a rendezvous point: threads call `pthread_barrier_wait` and block until a fixed number (`count`) have arrived, at which point all are released to continue. 

**The Concept:** You initialize a barrier with a specific count (e.g., 4). As threads finish their chunk of work, they call `pthread_barrier_wait()`. The first three threads to hit this function will instantly go to sleep. When the _fourth_ thread hits it, the barrier "breaks," and all four threads are immediately woken up to continue executing the next line of code simultaneously.


Barriers are useful for phased parallel algorithms (e.g., iterative solvers, simulation timesteps) where each phase must complete before the next begins. They are not a mutual exclusion mechanism: they do not protect shared data by themselves; they only coordinate *when* threads proceed.

**Step-by-step barrier cycle:**
- Each thread executes phase work.
- Each thread calls `pthread_barrier_wait`.
- Threads block until `count` threads have called it.
- The last arriving thread triggers release; exactly one thread gets `PTHREAD_BARRIER_SERIAL_THREAD`.
- All threads proceed to the next phase; the barrier is ready for reuse.


**Example (C/pthreads, two-phase computation):**
```c
#include <pthread.h>
#include <stdio.h>

#define N 4
static pthread_barrier_t bar;
static int partial[N];
static int total;

void *tfn(void *arg) {
    long id = (long)arg;

    partial[id] = (int)(id + 1);             // phase 1: compute partial
    int rc = pthread_barrier_wait(&bar);     // sync

    if (rc == PTHREAD_BARRIER_SERIAL_THREAD) {
        int sum = 0;
        for (int i = 0; i < N; i++) sum += partial[i];
        total = sum;                          // single thread publishes result
    }

    pthread_barrier_wait(&bar);               // ensure all see total after phase 2
    // now all threads can read 'total' safely (synchronized by barrier)
    return NULL;
}
```

This works because the barrier forces a "happens-before" style ordering: all writes to `partial[]` complete before the serial thread sums, and the second barrier ensures `total` is set before other threads proceed.



**Common mistakes / pitfalls:**
- Deadlock due to mismatched participants: if fewer than `count` threads reach the barrier (e.g., early return, error path), the rest block forever.
- Using a barrier when participant count is dynamic; barriers require a fixed count per cycle.
- Assuming a barrier provides mutual exclusion for concurrent updates; it does not.



        
## POSIX Semaphores (`sem_t`)


A POSIX semaphore is a counter with two core operations: **wait** (decrement, possibly block) and **post** (increment, possibly wake). 

Semaphores generalize locks: a binary semaphore (0/1) mimics a mutex-like gate, while a counting semaphore controls access to a pool of `K` identical resources.

- **Binary Semaphore (Max 1 token):** If the bucket only ever holds 1 token, it acts exactly like a mutex. Only one thread can be in the critical section at a time.
- **Counting Semaphore (Max K tokens):** If you have 5 database connections, you initialize the semaphore to 5. The first 5 threads to call `wait` get right through. The 6th thread is blocked until one of the first 5 calls `post`.
 

They are also commonly used for event signaling (e.g., producer-consumer), because they naturally represent "number of available items."

Internally, a semaphore maintains an integer value and a queue of waiters. `sem_wait` must atomically check and decrement; if the counter hits zero, it makes a system call, on Linux, this is usually a `futex` (Fast Userspace Mutex), telling the kernel, "Put me to sleep and take me off the CPU schedule until someone signals this specific memory address."

Semaphores do not enforce ownership: any thread can `sem_post` regardless of which thread performed `sem_wait`. This is the biggest distinction between a mutex and a semaphore, and it is a massive source of bugs. A semaphore has zero concept of ownership. Thread A can call `sem_wait`, and Thread B, Thread C, or even an entirely different process can call `sem_post`. This makes it incredibly powerful for the Producer-Consumer signaling mentioned above, but it means the compiler and the kernel won't protect you if your logic is flawed. If you accidentally write a loop that calls `sem_post` too many times, your counter inflates, your "locks" become meaningless, and multiple threads will crash into your critical section simultaneously.







### Named vs. Unnamed Semaphores


The core difference here is **where the semaphore lives** and **who can see it**.

- **Unnamed Semaphores (`sem_init`):** These are just variables (`sem_t`) living in your program's standard memory (heap or data segment). Because they are just memory addresses, they are incredibly fast. You use them to sync threads inside a single process.
    - _The Trap:_ If you fork a process, the child gets a _copy_ of that memory. The two processes will now be incrementing completely different semaphores. To share an unnamed semaphore across processes, you have to explicitly map it into shared memory (`mmap`). check [[virtual-memory]] for further explanation about cow.
    - Because they live right in your program's memory, they are perfect for managing **threads** that are running inside the _same_ program (intra-process). You don't need to ask the operating system to look up a name; they just work.

- **Named Semaphores (`sem_open`):** These are managed entirely by the OS kernel and act almost like files. You give them a string name (like `"/my_global_lock"`). Any completely unrelated process on the system can open that name and sync up.
    - _The Trap:_ Because the kernel manages them, they persist even if your C program crashes. If you don't explicitly call `sem_unlink()` to destroy the name, the next time you run your program, it will connect to the ghost of the old semaphore with whatever counter value was left behind.

	



### Wait and Post Operations

The API:

Think of a semaphore as a bucket of tickets. To do a task, a thread needs a ticket.
- `int sem_wait(sem_t *sem);`
	- The thread tries to take a ticket. If there are tickets (value > 0), it takes one (decrements) and keeps going. If the bucket is empty (value = 0), the thread **blocks** (goes to sleep) until a ticket becomes available.
    - _Note:_ It can be rudely awakened by a system signal (returning `-1` with `EINTR`)        
- `int sem_post(sem_t *sem);`
	- The thread puts a ticket back into the bucket (increments). If other threads are sleeping and waiting for a ticket, this wakes exactly _one_ of them up.
    
- `int sem_trywait(sem_t *sem);`
	- The thread tries to take a ticket. If the bucket is empty, instead of going to sleep, it immediately says "never mind" and returns an `EAGAIN` error so it can go do something else.
    
- `int sem_timedwait(sem_t *sem, const struct timespec *abs_timeout);`
	- The thread waits for a ticket, but only for a specific amount of time. If the clock runs out, it gives up and returns an `ETIMEDOUT` error.

The atomicity requirement is crucial: decrement-and-block must be indivisible to avoid missed wakeups. Implementations typically perform atomic decrement when possible and only enter the kernel when blocking/waking is required.


**Producer-consumer example (C/pthreads):**
```c
#include <pthread.h>
#include <semaphore.h>
#include <stdlib.h>

#define CAP 8
static int buf[CAP], head, tail;
static sem_t items, slots;
static pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;

void put(int x) {
    sem_wait(&slots);                 // wait for free slot
    pthread_mutex_lock(&m);
    buf[tail] = x;
    tail = (tail + 1) % CAP;
    pthread_mutex_unlock(&m);
    sem_post(&items);                 // signal available item
}

int get(void) {
    sem_wait(&items);                 // wait for available item
    pthread_mutex_lock(&m);
    int x = buf[head];
    head = (head + 1) % CAP;
    pthread_mutex_unlock(&m);
    sem_post(&slots);                 // signal free slot
    return x;
}
```

This code is solving a classic problem where one part of your program is creating data (Producer) and another part is processing it (Consumer), and they share a fixed-size queue (a ring buffer with 8 slots).

- **`slots` (Semaphore):** Tracks **empty spaces**. The Producer waits on this. If the buffer is full, the Producer sleeps until the Consumer frees up a slot.
    
- **`items` (Semaphore):** Tracks **full spaces** (actual data). The Consumer waits on this. If the buffer is empty, the Consumer sleeps until the Producer puts an item in.
    
- **Why the Mutex (`pthread_mutex_t m`)?** This is the most important takeaway. The semaphores only act as counters so the Producer doesn't overfill the buffer and the Consumer doesn't read from an empty one. But when a thread actually writes to `buf[tail]` or reads from `buf[head]`, it needs the mutex to lock the array. Without the mutex, two Producers could accidentally write to the exact same index at the exact same microsecond, corrupting your data.

**Common mistakes / pitfalls:**
- Using a semaphore as a mutex substitute but forgetting it has no ownership, enabling accidental double-`post` or missing `post`.
- Not handling `EINTR` from `sem_wait`: a signal can interrupt and the code must typically retry. Because `sem_wait` can be interrupted by system signals, you usually can't just call it once. You have to put it in a `while` loop so that if it gets interrupted, it immediately tries to wait again.
- Assuming `sem_post` wakes all waiters; it generally enables only as many waiters as the counter increase permits.

### Example

```c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

// This is our digital sign at the entrance
sem_t parking_spots;

void* car_behavior(void* car_id) {
    int id = *(int*)car_id;

    printf("Car %d arrived at the gate and is looking at the sign...\n", id);
    
    // 1. THE CAR TRIES TO ENTER (sem_wait)
    // If the semaphore is > 0, it decreases by 1 and the car enters.
    // If the semaphore is 0, the car blocks (sleeps) right here until a spot opens.
    sem_wait(&parking_spots); 
    
    printf("--> Car %d found a spot and parked!\n", id);
    
    // The car stays parked for 2 seconds
    sleep(2); 
    
    printf("<-- Car %d is leaving its spot.\n", id);
    
    // 2. THE CAR LEAVES (sem_post)
    // It increases the semaphore by 1, automatically waking up any car waiting at the gate.
    sem_post(&parking_spots); 
    
    return NULL;
}

int main() {
    // Set up our unnamed semaphore. 
    // The '0' means it's for threads in this program. 
    // The '3' is our starting value (3 parking spots available).
    sem_init(&parking_spots, 0, 3);
    
    pthread_t cars[5];
    int car_ids[5] = {1, 2, 3, 4, 5};

    // 5 cars arrive at the parking lot almost at the exact same time
    for (int i = 0; i < 5; i++) {
        pthread_create(&cars[i], NULL, car_behavior, &car_ids[i]);
    }

    // Wait for all 5 cars to finish their parking routines
    for (int i = 0; i < 5; i++) {
        pthread_join(cars[i], NULL);
    }

    // Tear down the parking lot
    sem_destroy(&parking_spots); 
    printf("Parking lot is closed.\n");
    
    return 0;
}
```


Here is why a semaphore is the perfect tool for the parking lot, and why a mutex would fail:

**The "Bouncer" vs. The "Bathroom Key"**

- **A Mutex is a Bathroom Key (Binary: 1 or 0):** A mutex is designed for strict _mutual exclusion_. It only allows **one** thread to access a resource at a time.
    
    - _If we used a mutex for the parking lot:_ Car 1 would grab the "key," enter the lot, and park. Even though there are two empty spots left, Cars 2 and 3 would be forced to wait at the gate until Car 1 left. It completely ruins the efficiency of having 3 spots.
        
- **A Semaphore is a Bouncer with a Clicker (Counting: N to 0):** A semaphore doesn't care _which_ spot a car takes, it only cares about the _total capacity_.
    
    - _Because we used a semaphore:_ We initialized it to 3 (`sem_init(&parking_spots, 0, 3);`). The bouncer lets Car 1 in (click: 2 left), Car 2 in (click: 1 left), and Car 3 in (click: 0 left) all at the same time. Only when the lot is actually full does the bouncer make Car 4 wait.
