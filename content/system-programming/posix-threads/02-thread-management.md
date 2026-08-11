---
title: "2. Thread Management"
---

part of the [[index|posix-threads]] series.

Thread management refers to the techniques and APIs involved in creating, manipulating, and controlling threads within a program. This includes starting new threads, altering their behavior, safely terminating them, and managing their lifecycle and resource handling. Proper thread management is crucial to ensure efficient parallel execution and avoid issues like memory leaks, deadlocks, or undefined behavior.


## Thread Creation (`pthread_create`)

To create a new thread, we use the **pthread_create()** function from the POSIX threads library. It launches a new thread that executes a specific function.

**Syntax:**
```c
int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine)(void *), void *arg);
```

where:
- **thread**: Pointer to a `pthread_t` variable where the thread identifier will be stored.
- **attr**: Pointer to a thread attributes object (`pthread_attr_t`). Use `NULL` to apply default attributes.
- **start_routine**: Pointer to the function to be executed. Must return `void*` and accept a single `void*` argument.
- **arg**: Passes an argument to the thread function. You can use a pointer to a struct or multiple values if needed.

**Example:**
```c
pthread_t thread;
pthread_create(&thread, NULL, my_function, (void *)argument);
```



## Thread Termination (`pthread_exit`, `pthread_cancel`)

### pthread_exit()

To terminate the current thread, call **pthread_exit()**. This cleans up any resources associated with the thread and returns a value that can be collected by a joining thread.

**Syntax:**
```c
void pthread_exit(void *retval);
```
- **retval**: Value returned to any thread joining this thread.

**Example:**
```c
pthread_exit((void *)result);
```

### pthread_cancel()
To request cancellation of another thread from outside, use **pthread_cancel()**. This sends a cancellation request which the target thread processes at defined cancellation points.

**Syntax:**
```c
int pthread_cancel(pthread_t thread);
```
- **thread**: Thread identifier of the target thread.

**Example:**
```c
pthread_cancel(thread);
```



## Joining and Detaching (`pthread_join`, `pthread_detach`)

### pthread_join()
To wait for a thread to finish and retrieve its exit status, use **pthread_join()**. This is essential for proper cleanup and synchronization.

**Syntax:**
```c
int pthread_join(pthread_t thread, void **retval);
```
- **thread**: Thread identifier to join.
- **retval**: Pointer to store the thread's exit value. Use `NULL` if not required.

**Example:**
```c
void *thread_result;
pthread_join(thread, &thread_result);
```

### pthread_detach()
To let a thread run independently without being joined later, call **pthread_detach()**. When detached, a thread's resources are automatically reclaimed after it finishes.

Use when you do not care about the thread's result and want to ensure its resources are freed immediately upon completion.

**Syntax:**
```c
int pthread_detach(pthread_t thread);
```
- **thread**: Thread identifier to detach.

**Example:**
```c
pthread_detach(thread);
```



## Thread Attributes and State

When creating threads, you can customize their behavior using a **pthread_attr_t** object. Thread attributes control properties such as stack size, scheduling policy, and whether the thread starts as detached or joinable.

it is the second argument while creating threads.

- `pthread_attr_init(&attr)`: Initializes the attributes object with default values.
- `pthread_attr_setstacksize(&attr, size)`: Sets the stack size for the thread.
- `pthread_attr_setdetachstate(&attr, state)`: Specifies whether the thread is created as joinable (`PTHREAD_CREATE_JOINABLE`) or detached (`PTHREAD_CREATE_DETACHED`).
- `pthread_attr_setschedpolicy(&attr, policy)`: Sets the scheduling policy for the thread.
- `pthread_attr_destroy(&attr)`: Cleans up the attributes object after use.

**Example:**
```c
pthread_t thread;
pthread_attr_t attr;

// Initialize attributes
pthread_attr_init(&attr);
pthread_attr_setstacksize(&attr, 1024 * 1024);               // Set stack size to 1 MB
pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE); // Set as joinable

// Create thread with attributes
pthread_create(&thread, &attr, my_function, (void *)arg);

// Destroy attribute object
pthread_attr_destroy(&attr);
```


## Thread Identification (`pthread_self`, `pthread_equal`)

To manage and distinguish threads in a multithreaded application, you can use **pthread_self()** to get the identifier of the current thread and **pthread_equal()** to compare thread identifiers. 

### pthread_self()

Returns the thread identifier of the calling thread. Useful for logging, debugging, or performing operations based on the thread's identity.

**Syntax:**
```c
pthread_t pthread_self(void);
```

**Example:**
```c
pthread_t tid = pthread_self();
// Now 'tid' uniquely identifies the calling thread
```

### pthread_equal()

Compares two thread identifiers to check if they refer to the same thread. This is needed because thread IDs are opaque objects and should not be compared directly with `==`.

it is possible for two `pthread_t` variables to refer to the same thread if they are obtained from separate calls but represent the same thread (example: you store the ID when the thread is created, and later use `pthread_self()` in the thread itself).

**Syntax:**
```c
int pthread_equal(pthread_t t1, pthread_t t2);
```

**Example:**
```c
if (pthread_equal(tid1, tid2)) {
    // Same thread
}
```
