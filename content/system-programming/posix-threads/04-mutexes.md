---
title: "4. Synchronization: Mutexes"
---

part of the [[index|posix-threads]] series.

In multithreaded systems, two or more threads often need to access shared resources or data structures. If this access is not synchronized properly, it can result in race conditions, data corruption, or inconsistent program state.

A **mutex** (mutual exclusion lock) is a low-level synchronization primitive used to enforce exclusive access to a resource:

- **Locking:** A thread acquires the mutex before accessing a critical section or shared data.
- **Unlocking:** The mutex is released when the thread is done, allowing others to proceed.
- Mutexes are simple, fast, and widely supported (e.g., POSIX `pthread_mutex_t`).


if multiple threads write or modify the same data we need to use a mutex



## Mutex Initialization and Destruction

When setting up a mutex, you must choose whether to initialize it statically (at declaration) or dynamically (at runtime). This decision affects flexibility, configuration options, memory management, and code clarity.


### Static Initialization

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
```

- **How:** Directly in declaration; no arguments or function calls.
- **Behavior:** Uses default type (`PTHREAD_MUTEX_NORMAL`), no custom attributes.
- **Cleanup:** No need for `pthread_mutex_destroy`.
- **When To Use:**
    - Mutex never needs special attributes.
    - Mutex exists for entire program life.
    - Simpler code, fewer errors.

### Dynamic Initialization

```c
pthread_mutex_t mutex;

pthread_mutexattr_t attr;

//attr init and set

pthread_mutex_init(&mutex, &attr);
// use mutex
pthread_mutex_destroy(&mutex);

```

- **How:** Initialize in code (after declaration), possibly with attributes.
- **Behavior:** Supports custom attributes (recursive, errorcheck, priority protocols).
- **Cleanup:** Must call `pthread_mutex_destroy(&mutex)`.
- **When To Use:**
    - Need recursive, errorchecking, or custom protocol.
    - Allocated mutex in heap/struct, not global/static.
    - Must control mutex lifespan (e.g., cleanup, dynamic threads).

Call this when you need custom attributes (recursive, error-check, etc.). Always pair with `pthread_mutex_destroy` for cleanup.

**Init Function:** 

```c
int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *attr);
```

- **Args:**
    - `mutex`: pointer to mutex object to initialize
    - `attr`: pointer to attribute object, or NULL for defaults

**Destroy Function:**

for freeing resources
```c
pthread_mutex_destroy(&mutex);
```


## Locking and Unlocking (`pthread_mutex_lock`, `pthread_mutex_unlock`)


**Locking with a mutex** means a thread claims exclusive access to a critical section or shared resource by acquiring a lock. Only the thread holding the mutex can access the protected data until it releases the lock (unlocking).


  ```c
  pthread_mutex_lock(&my_mutex);
  // critical section
  pthread_mutex_unlock(&my_mutex);
  ```
- **Failure to unlock leads to deadlock.**
- Only the locking thread should unlock.

**Syntax:**
```c
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```
- **Returns:** 0 on success; error code on failure.




## Non-blocking Locks (`pthread_mutex_trylock`)

A **non-blocking mutex lock** lets a thread attempt to lock a mutex, and immediately returns control if the mutex is already locked, rather than putting the thread to sleep.

it is useful when threads need to do work only if the resource is available, without waiting.

- **Syntax:**
```c
int pthread_mutex_trylock(pthread_mutex_t *mutex);
```

- **Returns:** 0 on success (lock acquired); `EBUSY` if already locked.

**Example:**
```c
if (pthread_mutex_trylock(&mutex) == 0) {
    // Safe to enter critical section
    // ... do work ...
    pthread_mutex_unlock(&mutex);
} else {
    // Mutex was busy; decide what to do instead
}
```

There's also a middle ground between `lock` (waits forever) and `trylock` (doesn't wait at all): `pthread_mutex_timedlock(mutex, abstime)` blocks like `lock`, but gives up and returns `ETIMEDOUT` if `abstime` (an absolute time) passes first.




## Mutex Attributes (Recursive, Errorcheck)

Attributes are options that change the behavior of a mutex, such as allowing the _same thread_ to relock (recursive) or checking for errors in lock/unlock usage (errorcheck).

the most common types of attr are:

| Type                       | Effect                                                                      | How set                 |
| -------------------------- | --------------------------------------------------------------------------- | ----------------------- |
| `PTHREAD_MUTEX_NORMAL`     | Standard mutex, no error checking; deadlocks if locked twice by same thread | Attribute, also default |
| `PTHREAD_MUTEX_RECURSIVE`  | Same thread can lock multiple times, must unlock same #                     | Attribute               |
| `PTHREAD_MUTEX_ERRORCHECK` | Mutex checks for errors: double lock, unlock by wrong thread                | Attribute               |

- **Recursive:** Allows complex or re-entrant code to lock the same mutex multiple times without deadlock (must unlock same number of times).
- **Errorcheck:** Detects mistakes like double-locks or unlocking a mutex not held by the thread, helping catch bugs early.

One more attribute worth knowing: `pthread_mutexattr_setpshared(&attr, PTHREAD_PROCESS_SHARED)` lets a mutex placed in shared memory (`mmap`) be locked across separate processes, not just threads in the same process. The same `pshared` attribute exists for condition variables and barriers too.

```c
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
```

### Set Mutex Type

**Function:**

```c
int pthread_mutexattr_settype(pthread_mutexattr_t *attr, int type);
```
- **Args:**
    - `attr`: The attribute object pointer
    - `type`: The mutex type:
        - `PTHREAD_MUTEX_NORMAL` (default)
        - `PTHREAD_MUTEX_RECURSIVE`
        - `PTHREAD_MUTEX_ERRORCHECK`
- Returns 0 on success.

**Example:**
```c
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
pthread_mutex_init(&mutex, &attr);
```


### Examples

```c
void log_message(int depth) {
	pthread_mutex_lock(&log_mutex);
	printf("Logging at depth %d\n", depth);

// Recursion: if depth > 0, call log_message again
	if (depth > 0) {
		log_message(depth - 1); // Locking same mutex again
	}

	pthread_mutex_unlock(&log_mutex);

}
```
in this example we use a recursive mutex and it lets us lock without unlocking the lock before. with a normal mutex this creates a deadlock.


```c
rc = pthread_mutex_unlock(&log_mutex); // Correct unlock

rc = pthread_mutex_unlock(&log_mutex); // Error: already unlocked

if (rc != 0) {
	printf("Second unlock failed (expected): %d\n", rc);
	if (rc == EPERM) {
		printf("Error: Unlocking mutex not owned by the thread (EPERM)\n");
	}
}
```
In this example, we use an **errorcheck** mutex and it lets us *detect incorrect usage*, such as unlocking a mutex that is not locked (or already unlocked) by the thread. With a normal mutex, this leads to undefined behavior. With errorcheck, `pthread_mutex_unlock` will return an error code if the operation is invalid.



## Priority Inversion and Mutex Protocols

Priority inversion occurs when a lower-priority thread holds a mutex needed by a higher-priority thread, and intermediate threads prevent the low-priority thread from ever running and releasing the lock.

Mutex protocols offer mechanisms (like _priority inheritance_) to avoid or mitigate this.



### Set Protocol

**Function:**

```c
int pthread_mutexattr_setprotocol(pthread_mutexattr_t *attr, int protocol);
```
- **Args:**
    - `attr`: Attribute object pointer
    - `protocol`: One of
        - `PTHREAD_PRIO_NONE` (default, no handling)
        - `PTHREAD_PRIO_INHERIT` (inherit blocking thread's priority)
        - `PTHREAD_PRIO_PROTECT`
- Returns 0 on success.

**Example:**

```c
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
pthread_mutex_init(&mutex, &attr);
```

### Types of Mutex Protocols

#### 1. `PTHREAD_PRIO_NONE` (Default)

- **No priority support.**
- System does nothing special.
- **Risk:** Priority inversion can happen, just like regular mutex.

#### 2. `PTHREAD_PRIO_INHERIT`

- **Prevents priority inversion.**
- If a higher-priority thread waits for the mutex, the lower-priority thread holding it temporarily "inherits" the higher priority, so it finishes sooner and releases the lock.
- Makes sure high-priority tasks aren't blocked unnecessarily.

#### 3. `PTHREAD_PRIO_PROTECT`

- **Priority ceiling protocol.**
- Each mutex is given a fixed "ceiling" priority.
- When a thread locks the mutex, its priority is boosted to this ceiling as long as it holds the lock.
- Used for strict real-time systems with precise control needs.
