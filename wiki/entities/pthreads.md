---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [advanced-linux-programming, understanding-the-linux-kernel]
tags: [threads, pthreads, concurrency, synchronization, posix]
---

# POSIX Threads (pthreads)

POSIX threads provide a standardized API for creating and managing threads within a single process. On Linux, each pthread maps 1:1 to a kernel lightweight process (a `task_struct` created with `CLONE_VM | CLONE_FILES | CLONE_FS | CLONE_SIGHAND | CLONE_THREAD`), meaning the kernel schedules threads individually. The user-space API is provided by the NPTL (Native POSIX Thread Library) in glibc, linked with `-lpthread`.

## Thread Lifecycle

### Creation

```c
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                   void *(*start_routine)(void *), void *arg);
```

Creates a new thread executing `start_routine(arg)`. The `attr` parameter controls stack size, scheduling policy, and detach state (NULL uses defaults). The new thread begins executing immediately (it may run before `pthread_create` returns).

### Joining and Detaching

- **`pthread_join(thread, &retval)`** — blocks the caller until the target thread terminates. Retrieves the thread's return value. Analogous to `waitpid()` for processes. Each joinable thread must be joined exactly once to reclaim its resources.
- **`pthread_detach(thread)`** — marks a thread as detached. Its resources are automatically reclaimed on termination (no `pthread_join` needed or permitted). A thread can also be created detached via `pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED)`.

### Termination

A thread terminates by:
1. Returning from its start routine (return value is available to `pthread_join`).
2. Calling `pthread_exit(retval)` explicitly.
3. Being cancelled via `pthread_cancel()`.
4. Any thread in the process calling `exit()` (terminates all threads).

## Thread Cancellation

`pthread_cancel(thread)` requests cancellation of a target thread. The actual cancellation depends on the target's cancellation state and type:

| Setting | Function | Values | Default |
|---------|----------|--------|---------|
| **State** | `pthread_setcancelstate()` | `PTHREAD_CANCEL_ENABLE` / `PTHREAD_CANCEL_DISABLE` | Enabled |
| **Type** | `pthread_setcanceltype()` | `PTHREAD_CANCEL_DEFERRED` / `PTHREAD_CANCEL_ASYNCHRONOUS` | Deferred |

With **deferred cancellation** (the default and recommended mode), the thread is only cancelled when it reaches a **cancellation point** — a POSIX-defined set of functions that include `pthread_join()`, `read()`, `write()`, `sleep()`, `sem_wait()`, `pthread_cond_wait()`, and many others.

### Cleanup Handlers

```c
pthread_cleanup_push(cleanup_function, arg);
/* ... critical section ... */
pthread_cleanup_pop(execute);  /* execute=1 to call handler, 0 to just remove */
```

Cleanup handlers are called (in LIFO order) when a thread is cancelled or calls `pthread_exit()`. They serve the same purpose as `finally` blocks — releasing mutexes, freeing memory, closing files. Push/pop calls must be lexically paired (they are implemented as macros that open/close a brace).

## Thread-Specific Data

Provides per-thread global variables — useful for making libraries thread-safe without changing APIs:

```c
pthread_key_t key;
pthread_key_create(&key, destructor_fn);  /* once, typically in pthread_once */
pthread_setspecific(key, value);           /* per thread */
void *val = pthread_getspecific(key);      /* per thread */
```

The optional `destructor_fn` is called automatically with the thread-specific value when the thread terminates, enabling automatic cleanup. This mechanism underlies the `errno` variable in multi-threaded programs — each thread has its own `errno`.

## Synchronization Primitives

### Mutexes

The fundamental locking primitive for protecting shared data:

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_lock(&mutex);     /* blocks until lock acquired */
/* ... critical section ... */
pthread_mutex_unlock(&mutex);
```

- **`pthread_mutex_trylock()`** — non-blocking variant, returns `EBUSY` if the mutex is held.
- **Deadlock prevention** — always acquire multiple mutexes in a consistent global order. Use `trylock()` for back-off strategies.
- **Mutex types**: normal (default — undefined behavior on re-lock by same thread), error-checking (returns `EDEADLK`), recursive (allows re-locking by same thread with a count).

Mutexes on Linux are implemented atop the **futex** (fast userspace mutex) mechanism — uncontended lock/unlock operates entirely in user space with atomic compare-and-swap, only entering the kernel when contention requires sleeping.

### Semaphores

POSIX unnamed semaphores — counting semaphores for signaling and resource counting:

```c
sem_t sem;
sem_init(&sem, 0, initial_value);  /* pshared=0 for intra-process */
sem_wait(&sem);   /* decrement; blocks if value is 0 */
sem_post(&sem);   /* increment; wakes one waiter */
sem_destroy(&sem);
```

Unlike mutexes, semaphores have no ownership — any thread can post (increment). `sem_wait()` decrements, blocking if the count would go negative. `sem_trywait()` is the non-blocking variant. `sem_getvalue()` reads the current count.

With `pshared=1`, a semaphore placed in shared memory can synchronize threads across different processes.

### Condition Variables

Enable threads to wait for arbitrary conditions, not just lock availability:

```c
pthread_mutex_lock(&mutex);
while (!condition)
    pthread_cond_wait(&cond, &mutex);  /* atomically unlocks mutex + sleeps */
/* condition is true, mutex is held */
pthread_mutex_unlock(&mutex);
```

- **`pthread_cond_signal()`** — wakes at least one waiting thread.
- **`pthread_cond_broadcast()`** — wakes all waiting threads.
- The `while` loop around `pthread_cond_wait()` is mandatory — **spurious wakeups** can occur, and the condition must be re-checked after every wakeup.
- `pthread_cond_timedwait()` adds an absolute-time deadline.

The mutex+condition variable pattern is the building block for producer-consumer queues, barriers, read-write locks, and other higher-level synchronization constructs.

## Thread vs. Process Trade-offs

| Aspect | Threads (pthreads) | Processes (fork) |
|--------|-------------------|------------------|
| Address space | Shared — all threads see all memory | Separate — COW copy, isolated |
| Communication | Direct memory access (with synchronization) | Requires IPC (pipes, shmem, sockets) |
| Creation cost | Lower (no page table copy) | Higher (COW setup, descriptor copy) |
| Fault isolation | A crash in one thread kills all threads | A crash in one process leaves others running |
| Debugging | Harder (shared state, race conditions) | Easier (isolated state) |
| Scalability | Limited by single address space | Each process has full address space |

General guidance: use threads when you need low-latency communication between concurrent tasks and can manage shared-state complexity. Use processes when you need fault isolation or are wrapping independent programs via `exec()`.

## Relationship to Kernel Internals

At the kernel level, each pthread is a lightweight process scheduled by the [process-scheduler](process-scheduler.md). Thread synchronization primitives use the **futex** system call for efficient user-kernel transitions. The kernel's [concept-kernel-synchronization](../concepts/concept-kernel-synchronization.md) primitives (spinlocks, RCU, etc.) are distinct from pthread primitives — they are for kernel-space use only.

Signal delivery in multi-threaded programs follows the rules described in [signals](signals.md): process-directed signals go to any thread with the signal unblocked; thread-directed signals (e.g., `SIGSEGV` from a memory fault) go to the faulting thread.

## See also

- [process-management](process-management.md)
- [signals](signals.md)
- [concept-kernel-synchronization](../concepts/concept-kernel-synchronization.md)
- [system-calls](system-calls.md)
