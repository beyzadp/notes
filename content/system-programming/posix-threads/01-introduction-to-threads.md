---
title: "1. Introduction to Threads"
---

part of the [[index|posix-threads]] series.

## Concurrency vs. Parallelism

**Concurrency** is the ability of a system to make progress on multiple tasks at once by interleaving their execution, often through techniques like context switching or running tasks on separate threads. It improves resource utilization and responsiveness, but does not necessarily guarantee that tasks are executing simultaneously. 

If there are two processes that both want to read and then write data, concurrency lets them make progress at the same time by taking turns using the CPU. For example, while Process 1 is waiting to read, Process 2 can start its own reading, and then Process 1 can write while Process 2 waits. so their actions overlap instead of happening one after the other. This overlap makes the system more efficient, since the CPU isn't idle while a process waits for slow operations like reading or writing.

On the other hand **parallelism** is the simultaneous execution of multiple tasks or processes to solve a problem faster. Instead of processing data one piece at a time (sequentially), a large task is broken down into smaller, independent parts that are processed at the **exact same moment** using multiple CPU cores or processors.

For example, while Process 1 is reading on one core, Process 2 is also reading on another core simultaneously, and later they both write their results at the same time. This allows their actions to truly happen side by side, making tasks finish faster when hardware allows it.

**Parallelism** is different from **concurrency**: Parallelism is about doing multiple things at exactly the same time, while concurrency is about managing multiple tasks, which may or may not actually run simultaneously.

## Threads vs. Processes

A process is an independent program that is running on your computer. It has its own memory space (heap, stack), resources, and address space. 

a process
- Has its own **address space**.
- Contains resources like file handles, registers, memory page tables.
- Communication between processes usually happens using **Inter-Process Communication (IPC)** mechanisms (e.g. pipes, sockets).
- **Heavyweight**: Creating a new process involves duplicating resources (expensive for the system).
- If a process crashes, it does not directly affect other processes.

A thread is a smaller unit of execution within a process. A process can have multiple threads (multithreading), which share the same resources and address space. 

a thread
- Shares the **address space** and resources with other threads in the same process. that means all threads in a process can access the same variables, data structures, and memory locations (heap section). if a program has a variable, every thread in the same process can read or modify it.
- Has its own **registers**, **stack**, and **program counter**.
```
Process memory:
    Heap (shared)
    Stack for Thread 1 (local)
    Stack for Thread 2 (local)
    Stack for Thread 3 (local)
```
- Communication between threads is easier (since memory is shared).
- **Lightweight**: Creating a thread is less expensive as it uses the shared resources of the process.
- If a thread crashes, it may affect other threads (since they share the same memory).


When multiple threads run simultaneously, they might try to read or modify the same data at the same time, causing a race condition. A **lock** makes sure that only one thread can access the critical section (the piece of code that manipulates shared data) at a time.

A **lock** (or mutex, which stands for "mutual exclusion") is a synchronization primitive used in multithreading programming to control access to shared resources, like variables, data structures, or sections of memory.


## Benefits and Risks of Multithreading

### Benefits

Multithreading allows a process to spawn multiple threads of execution. On multi-core CPUs, threads can be scheduled to run concurrently on separate cores, maximizing the utilization of the CPU. This increases overall system throughput because multiple instructions can be executed simultaneously. 

On systems with symmetric multiprocessing (SMP) or multi-core architectures, multithreading enables software to scale with hardware advances by executing different threads in parallel, thereby reducing overall runtime for parallelizable workloads.

In interactive or real-time applications, using multiple threads ensures that time-consuming tasks (such as disk I/O, network communication, or heavy computation) do not block the main (UI) thread. This lets the main event loop or message handler continue responding to user events, reducing latency and preventing the application from becoming unresponsive.

Threads within the same process have access to the same heap, global variables, and file descriptors. This facilitates low-overhead, fast inter-thread communication and reduces memory usage compared to using multiple processes, which do not share address space.



### Risks

A race condition occurs when two or more threads access shared data and at least one thread modifies it, resulting in undefined or inconsistent data if proper synchronization (such as with a mutex or spinlock) is not used. 

Deadlock arises when two or more threads form a cyclic dependency by each holding a lock and waiting indefinitely for another lock held by one of the other threads. Classic deadlock scenarios include nested lock acquisition or circular waiting patterns.
For example if Thread A locks MutexX and waits for MutexY, while Thread B locks MutexY and waits for MutexX.

**Livelock** happens in multithreading when two or more threads are continuously changing their state in response to each other, but none of them are able to actually finish their work or reach their goal. Unlike **deadlock** (where threads are stuck waiting and do nothing), in a **livelock**, threads are not blocked, they keep running and reacting but with no forward progress.

Locks (mutex, semaphore, condition variable, etc.) incur CPU and memory overhead to manage critical sections and avoid race condition. High contention or frequent context switches can degrade multithreaded performance and diminish the benefits of concurrency.

Multithreaded programs exhibit non-deterministic execution (different thread interleavings produce different results), making bugs sporadic and hard to reproduce. Diagnosing and debugging issues like race condition or deadlock often requires specialized tools (e.g., thread sanitizer, race detector).

The OS must manage context for each thread (registers, stack, program counter, etc.). Frequent context switches between threads introduce additional CPU overhead, potentially lowering overall throughput and causing cache pollution.

Different operating systems and language runtimes implement threading models differently (POSIX thread (pthread) on Unix-like systems, Windows thread API, Java Virtual Machine threads). Semantics, scheduling, and supported thread operations can vary, impacting portability and predictability.




## The POSIX Thread (Pthread) Standard

The **POSIX Thread (Pthread) Standard** defines a **multithreading API** for C programs, allowing you to create, control, synchronize, and communicate between threads within a process. It's part of the IEEE POSIX 1003.1c specification, intended to ensure **portability** and **compatibility** across UNIX-like operating systems.

Pthread exists so we don't have to struggle with different OS thread models, letting us write portable, reliable multithreaded code with guaranteed features, something not possible (or practical) with custom libraries and raw system calls alone. 


- **API** is a set of functions and constants for thread management, synchronization, and interaction.

---

### Major Components of the Pthread API

| Component         | Purpose                                              | Example Functions                                                  |
| ----------------- | ---------------------------------------------------- | ------------------------------------------------------------------ |
| Thread Management | Create, join, detach, exit threads                   | `pthread_create`, `pthread_join`, `pthread_exit`, `pthread_detach` |
| Attribute Objects | Set thread properties (stack size, scheduling, etc.) | `pthread_attr_init`, `pthread_attr_setdetachstate`                 |
| Synchronization   | Coordinate thread execution                          | `pthread_mutex_t`, `pthread_cond_t`, `pthread_rwlock_t`            |
| Communication     | Share and protect data between threads               | Mutex, Condition Variable                                          |

---

### Example: Thread Creation

```c
#include <pthread.h>
#include <stdio.h>

void* thread_function(void* arg) {
    printf("Hello from thread!\n");
    return NULL;
}

int main() {
    pthread_t thread;
    pthread_create(&thread, NULL, thread_function, NULL);
    pthread_join(thread, NULL);
    return 0;
}
```

- `pthread_t`: Thread identifier (thread handle)
- `pthread_create`: Starts a new thread executing `thread_function`
- `pthread_join`: Waits for thread to finish

---

### Synchronization Example: Mutex

```c
pthread_mutex_t lock;

void* safe_increment(void* arg) {
    pthread_mutex_lock(&lock);
    // critical section
    pthread_mutex_unlock(&lock);
    return NULL;
}
```

- **mutex (pthread_mutex_t)**: Prevents concurrent access to shared resources ("critical section")
