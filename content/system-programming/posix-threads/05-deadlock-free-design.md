---
title: "5. Designing Deadlock-Free Code"
---

part of the [[index|posix-threads]] series.

Deadlock happens when two or more threads get stuck forever because each one is waiting for another to release a resource (usually a `mutex`/`lock`). Designing deadlock-free code means structuring how threads acquire and release locks so they can't form "waiting cycles".

Typical symptoms:

- A program "hangs" (no progress) even though CPU usage might be low.
- Threads are blocked in `lock()` calls.
- Reproduces only sometimes (timing-dependent).

## The Four Coffman Conditions

Deadlock requires *all four* of these conditions to be true at the same time. If you break **any one** of them, deadlock cannot occur.

### 1) Mutual Exclusion

At least one resource must be held in a non-shareable way (only one thread can own it at a time).

- Example: a `mutex` protects a critical section; only one thread can hold the lock.

Most locks are inherently mutually exclusive, so this condition is often unavoidable.

---

### 2) Hold and Wait

A thread holds at least one resource while waiting to acquire additional resources.

- Example: Thread A holds `lock1`, then tries to acquire `lock2` without releasing `lock1`.

This is one of the easiest conditions to reduce with careful design.

---

### 3) No Preemption
Resources cannot be forcibly taken away; they must be released voluntarily by the holding thread.

- Example: If a thread holds a `mutex`, another thread cannot "steal" it; it must wait.

Most `mutex` implementations have no preemption by design.

---

### 4) Circular Wait
There exists a cycle of threads where each thread waits for a resource held by the next thread in the cycle.

- Example:  
  - Thread A holds `lock1` and waits for `lock2`  
  - Thread B holds `lock2` and waits for `lock1`  
  This forms a cycle: A -> B -> A

Preventing circular wait is a very common strategy (via lock ordering).


### A Classic Deadlock Example 

Two locks: `A` and `B`

```c
// Thread 1
lock(A);
lock(B);   // waits here if Thread 2 holds B
// ...
unlock(B);
unlock(A);

// Thread 2
lock(B);
lock(A);   // waits here if Thread 1 holds A
// ...
unlock(A);
unlock(B);
```

If Thread 1 gets `A` and Thread 2 gets `B` first, both can wait forever: deadlock.

## Lock Ordering and Hierarchy

### Core Idea

Define a **global order** for acquiring locks (a hierarchy), and require *everyone* to acquire them in that order. This prevents **circular wait**.

Example global order:

1. `lock_user`
2. `lock_account`
3. `lock_transaction`

Rule: if a thread needs multiple locks, it must acquire them in increasing order: `lock_user` -> `lock_account` -> `lock_transaction`.



**How to Implement Lock Hierarchy in Practice**

Common approaches:

- **Documented convention**: a written rule (works best with code reviews).
- **Lock levels**: assign each lock a rank/level; assert that a thread never acquires a lower-rank lock after a higher-rank lock.
- **Use combined locking utilities** (where available): functions that lock multiple locks safely.

### Example: Fixing the Deadlock with Ordering

If we decide the order is `A` then `B`, **no code is allowed to lock `B` then `A`**.

```c
// Thread 1
lock(A);
lock(B);
// ...
unlock(B);
unlock(A);

// Thread 2 (must follow same order)
lock(A);
lock(B);
// ...
unlock(B);
unlock(A);
```

Now, Thread 2 can't hold `B` while waiting for `A`, so the cycle cannot form.


## Deadlock Avoidance Strategies

These are techniques to *reduce risk* (and often improve overall concurrency). Some prevent deadlock by breaking Coffman conditions; others make deadlocks very unlikely or recoverable.

### 1) `try_lock` + Back-off (Avoid waiting forever)
Instead of blocking on a lock, attempt acquisition and back off if it fails. This helps avoid "hold and wait".

**Pattern:**
1. Acquire first lock
2. `try_lock` the second
3. If it fails, release first lock, wait a bit, retry

```c
while (true) {
    lock(A);
    if (try_lock(B)) {
        // acquired both
        break;
    }
    unlock(A);

    sleep_random_short_time(); // back-off to reduce contention
}

// critical section using A and B

unlock(B);
unlock(A);
```

**Pros**
- Avoids threads waiting forever while holding locks.
- Can improve responsiveness under contention.

**Cons**
- Must be careful about starvation (a thread might retry forever).
- More complex control flow.

---

### 2) Reduce Lock Granularity (or restructure shared state)
If one `mutex` protects too much, many threads contend and may need multiple locks at once. Consider:

- Splitting one big lock into multiple smaller locks (per object / per bucket).
- Reducing the time spent holding a lock.
- Moving work outside the critical section.

**Example idea**
- Instead of one global `lock_all_accounts`, use `lock_account[id]`.

**Tradeoff:** Finer granularity can increase complexity and the chance you need *multiple* locks, so combine with **lock ordering** (e.g., lock accounts by ascending ID).

---

### 3) Avoid Nested Locks (Break "Hold and Wait")
Design so that you rarely hold one lock while acquiring another. Tactics:

- Copy shared data under a lock, then release lock and process the copy.
- Use message passing / queues (one dedicated thread owns the data).
- Use "open calls": don't call unknown/external functions while holding a lock.

**Why "open calls" matter:** If you call into code that might acquire another lock (or call back into you), you can create accidental lock cycles.

---

### 4) Timeouts (Detect and Recover)
Use timed lock acquisition: if a lock can't be acquired within a time limit, the thread can log, rollback, release locks, or retry.

- Example concept: `try_lock_for(50ms)` (language/library dependent)

**Pros**
- Helps detect issues in production.
- Prevents infinite waiting.

**Cons**
- Doesn't guarantee correctness by itself; you need a recovery path.

---

### 5) Use Higher-Level Concurrency Tools Where Possible
Sometimes you can avoid manual lock management:

- Thread-safe data structures
- `RWLock` (reader-writer lock) when reads dominate
- Immutable data + swap pointer under one lock
- Actor/message model (one thread owns state)

These reduce the need for complex multi-lock protocols.


### Summary

- **Always** define and follow a lock order when multiple locks can be taken.
- Keep critical sections **short**.
- Avoid calling external/unknown functions while holding a lock.
- If you must lock multiple "same-type" objects, lock them by a stable key (e.g., **ascending ID**).
- Consider `try_lock` + back-off or timeouts in high-contention paths.
- Add debug tooling: log lock wait times; enable deadlock detection when available.
