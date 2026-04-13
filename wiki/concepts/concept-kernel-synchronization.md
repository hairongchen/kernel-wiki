---
type: concept
created: 2026-04-08
updated: 2026-04-08
sources: [understanding-the-linux-kernel]
tags: [synchronization, locking, concurrency, smp]
---

# Kernel Synchronization

Kernel synchronization is the set of mechanisms that protect shared data from concurrent access by multiple CPUs (SMP), interrupt handlers, softirqs, and preemptible kernel threads. Linux 2.6 provides a rich hierarchy of primitives, from lock-free per-CPU variables to sleeping semaphores. Choosing the right primitive is critical for both correctness and performance.

The fundamental sources of concurrency in the kernel are:

1. **SMP parallelism** — multiple CPUs executing kernel code simultaneously
2. **Interrupt handlers** — can preempt kernel code at almost any point
3. **Softirqs and tasklets** — deferred interrupt processing runs between system calls and after interrupts
4. **Kernel preemption** — since Linux 2.6, kernel code can be preempted at any point where preemption is enabled

## 1. Per-CPU Variables

The lightest synchronization mechanism is to avoid sharing altogether. **Per-CPU variables** give each CPU its own private copy of a data item. Since only one CPU ever accesses its own copy, no locking is needed.

```c
DEFINE_PER_CPU(type, name);          /* declaration */
per_cpu(name, cpu)                    /* access specific CPU's copy */
get_cpu_var(name)                     /* access current CPU's copy + disable preemption */
put_cpu_var(name)                     /* re-enable preemption */
```

The `get_cpu_var()` / `put_cpu_var()` pair disables preemption around the access to prevent the task from being migrated to another CPU mid-access. Per-CPU variables are used extensively in the networking stack, scheduler, and memory allocator.

**Limitations**: Only useful when CPUs do not need to read each other's data. If a global view is needed, the values must be aggregated explicitly.

## 2. Atomic Operations

For simple integer counters and bit flags that need to be updated atomically, the kernel provides `atomic_t` (and `atomic64_t` on 64-bit):

```c
atomic_t v = ATOMIC_INIT(0);
atomic_read(&v);                /* read */
atomic_set(&v, n);              /* write */
atomic_inc(&v);                 /* v++ */
atomic_dec(&v);                 /* v-- */
atomic_add(i, &v);              /* v += i */
atomic_sub(i, &v);              /* v -= i */
int atomic_dec_and_test(&v);    /* --v == 0 ? returns true if zero */
int atomic_inc_and_test(&v);    /* ++v == 0 ? */
int atomic_sub_and_test(i, &v); /* (v -= i) == 0 ? */
int atomic_add_negative(i, &v); /* (v += i) < 0 ? */
```

On x86, these are implemented with the `LOCK` prefix on the relevant instruction (e.g., `lock incl`), which locks the memory bus (or cache line) for the duration of that single instruction.

**Atomic bit operations**:
```c
set_bit(nr, addr);              /* set bit nr */
clear_bit(nr, addr);            /* clear bit nr */
test_and_set_bit(nr, addr);     /* set bit, return old value */
test_and_clear_bit(nr, addr);   /* clear bit, return old value */
```

Atomic operations are fast (no context switch, no spinlock overhead) but only protect single variables. They do not provide ordering guarantees between different memory locations — for that, memory barriers are needed.

## 3. Memory Barriers

Modern CPUs and compilers may reorder memory accesses for performance. When the order of reads and writes to different memory locations matters (e.g., setting a flag after writing data), explicit **memory barriers** are required:

| Barrier | Effect |
|---------|--------|
| `mb()` | Full memory barrier — all loads and stores before the barrier complete before any loads or stores after it |
| `rmb()` | Read barrier — orders reads only |
| `wmb()` | Write barrier — orders writes only |
| `smp_mb()`, `smp_rmb()`, `smp_wmb()` | SMP variants — expand to barriers on SMP, no-ops on UP (avoids unnecessary overhead) |
| `barrier()` | **Compiler barrier** only — prevents the compiler from reordering across this point, but does not emit any CPU fence instruction. Used when compiler reordering is the concern, not CPU reordering |
| `read_barrier_depends()` | Orders only loads with data dependencies (on most architectures, this is a no-op since data-dependent loads are naturally ordered — needed only on Alpha) |

### Example: Producer-Consumer Pattern

```c
/* Producer (CPU 0) */
data = new_value;     /* write data */
wmb();                /* ensure data is visible before flag */
flag = 1;             /* signal that data is ready */

/* Consumer (CPU 1) */
if (flag) {           /* read flag */
    rmb();            /* ensure flag is read before data */
    use(data);        /* read data */
}
```

Without the barriers, the CPU or compiler might reorder the writes (producer stores flag before data) or reads (consumer reads data before flag), leading to the consumer seeing stale data.

## 4. Spinlocks

Spinlocks are the fundamental mutual-exclusion primitive for short critical sections where sleeping is not allowed (interrupt context, softirq context, or when holding other spinlocks).

A spinlock is a simple integer: 1 = unlocked, 0 (or negative) = locked. Acquisition is an atomic test-and-set loop. On uniprocessor kernels with preemption, spinlocks degenerate to preemption disable/enable.

### Variants

| Function | Disables | Use when... |
|----------|----------|-------------|
| `spin_lock(&lock)` | Preemption | The data is accessed only from process context (not from interrupts or softirqs) |
| `spin_lock_irq(&lock)` | Preemption + hardware interrupts | The data is shared between process context and a hardware interrupt handler |
| `spin_lock_irqsave(&lock, flags)` | Preemption + hardware interrupts (saves/restores IF flag) | Same as `_irq`, but safe to use when the caller does not know if interrupts are already disabled |
| `spin_lock_bh(&lock)` | Preemption + softirqs (bottom halves) | The data is shared between process context and softirq/tasklet handlers, but not hardware interrupt handlers |

The matching unlock functions are `spin_unlock()`, `spin_unlock_irq()`, `spin_unlock_irqrestore()`, and `spin_unlock_bh()`.

**Critical rules**:
- Never sleep while holding a spinlock (the CPU would spin forever waiting for itself).
- Keep critical sections short — other CPUs waiting for the lock are burning CPU cycles.
- Acquire locks in a consistent order to avoid deadlocks.

## 5. Read-Write Spinlocks

When data is read frequently but written rarely, `rwlock_t` allows concurrent readers:

```c
rwlock_t lock = RW_LOCK_UNLOCKED;
read_lock(&lock);     /* multiple readers can hold this simultaneously */
/* read data */
read_unlock(&lock);

write_lock(&lock);    /* exclusive — waits for all readers and writers to release */
/* modify data */
write_unlock(&lock);
```

The same `_irq`, `_irqsave`, and `_bh` variants exist. Read-write spinlocks are useful when readers greatly outnumber writers, but they have a weakness: writers can be starved if a continuous stream of readers arrives. Seqlocks address this.

## 6. Seqlocks

Seqlocks are a lightweight mechanism optimized for data that is rarely written but read very frequently, and where it is acceptable for readers to retry:

```c
seqlock_t lock = SEQLOCK_UNLOCKED;

/* Writer (exclusive, like a spinlock) */
write_seqlock(&lock);
/* modify data */
write_sequnlock(&lock);

/* Reader (optimistic, lock-free) */
unsigned int seq;
do {
    seq = read_seqbegin(&lock);
    /* read data — may be inconsistent if a writer is active */
} while (read_seqretry(&lock, seq));
```

Internally, a seqlock contains a spinlock (for writer exclusion) and a sequence counter. Writers increment the counter before and after modification (making it odd during the write). Readers check the counter before and after reading — if it changed, or is odd, they retry.

**Key advantage over rwlocks**: Writers never wait for readers. This is critical for `xtime` (see [timing-subsystem](../entities/timing-subsystem.md)), which must be updated on every timer tick without delay, even if many CPUs are simultaneously reading the time.

**Limitation**: Readers may see inconsistent data during a write and must not dereference pointers read under the seqlock (the pointer target might be freed mid-read). Seqlocks protect simple value types, not pointer-chased structures.

## 7. Read-Copy-Update (RCU)

RCU is the most scalable read-side synchronization primitive. It allows **readers to proceed without any locking, atomic operations, or memory barriers**, at the cost of more complex update logic.

### Read Side

```c
rcu_read_lock();      /* = preempt_disable() — almost zero cost */
/* read RCU-protected data via rcu_dereference(ptr) */
rcu_read_unlock();    /* = preempt_enable() */
```

RCU readers cannot be preempted, but they can overlap with writers. There is no cache-line bouncing, no atomic instructions, no memory barriers — this is why RCU scales perfectly to any number of CPUs.

### Write / Update Side

RCU updates follow a publish-subscribe pattern:

1. **Copy**: Allocate a new version of the data structure.
2. **Update**: Modify the new copy.
3. **Publish**: Replace the old pointer with `rcu_assign_pointer(ptr, new)` (includes a write barrier).
4. **Wait**: Call `synchronize_rcu()` (blocking) or `call_rcu(callback)` (non-blocking) to wait for a **grace period** — the point at which all CPUs have gone through a context switch or returned to user mode, guaranteeing no reader still holds a reference to the old data.
5. **Free**: After the grace period, free the old data.

### Grace Period

A grace period ends when every CPU has been observed in a quiescent state (not in an RCU read-side critical section). In 2.6.11, this is detected by watching for context switches on each CPU. Since `rcu_read_lock()` is just `preempt_disable()`, a context switch implies the CPU is not in a read-side critical section.

RCU is used extensively in the kernel for routing tables, dcache lookups, module lists, and other read-mostly data structures. See also [interrupt-handling](../entities/interrupt-handling.md) for how RCU relates to softirq processing.

## 8. Semaphores

Semaphores are **sleeping locks** — if the lock is contended, the acquiring task sleeps instead of spinning. This makes them appropriate for long critical sections or critical sections that may need to perform blocking I/O.

```c
struct semaphore sem;
sema_init(&sem, 1);              /* initialize as mutex (count = 1) */

down(&sem);                       /* acquire (sleep if unavailable, uninterruptible) */
down_interruptible(&sem);         /* acquire (sleep if unavailable, return -EINTR on signal) */
down_trylock(&sem);               /* try to acquire, return nonzero if unavailable */
up(&sem);                         /* release */
```

Unlike spinlocks, semaphores can be held while sleeping. Unlike mutexes (which were introduced later), the semaphore's count can be initialized to values greater than 1, allowing it to function as a **counting semaphore** (e.g., limiting concurrency to N).

**down_interruptible()** is the preferred acquisition function for code called from user-space paths, because it allows the process to be interrupted by a signal (the caller must check the return value and handle `-ERESTARTSYS`).

## 9. Read-Write Semaphores

Analogous to read-write spinlocks, but sleep-capable:

```c
struct rw_semaphore rw_sem;
init_rwsem(&rw_sem);

down_read(&rw_sem);      /* shared access — multiple readers allowed */
up_read(&rw_sem);

down_write(&rw_sem);     /* exclusive access */
up_write(&rw_sem);

downgrade_write(&rw_sem); /* atomically convert write lock to read lock */
```

Read-write semaphores are used for `mm_struct.mmap_sem` (protects the process address space — taken for read on page faults, for write on `mmap`/`munmap`).

## 10. Completions

Completions are a synchronization mechanism for the pattern "wait until some other context has finished something":

```c
DECLARE_COMPLETION(done);

/* Waiting side */
wait_for_completion(&done);       /* sleeps until complete() is called */

/* Signaling side */
complete(&done);                  /* wake one waiter */
complete_all(&done);              /* wake all waiters */
```

Completions are used instead of semaphores when the semantics are "signal once" rather than "mutual exclusion." Common uses include waiting for kernel thread startup, disk I/O completion, and module unload synchronization. They handle edge cases (the signal arriving before the wait) correctly, which is tricky to do with semaphores.

## 11. Big Kernel Lock (BKL)

The **Big Kernel Lock** (`lock_kernel()` / `unlock_kernel()`) is a legacy recursive spinlock from Linux 2.0 — the original SMP serialization mechanism. It is recursive, auto-released on `schedule()`, and global (one lock for the entire kernel). By 2.6.11, most subsystems had replaced BKL with fine-grained locking; it remained mainly in legacy filesystem code and ioctl paths. Avoid in new code.

## Kernel Preemption

Linux 2.6 supports **kernel preemption**: a process running in kernel mode can be involuntarily preempted (switched out) if a higher-priority task becomes runnable. This improves latency for interactive and real-time workloads but introduces additional synchronization requirements.

### preempt_count

Each thread's `thread_info` contains a `preempt_count` field that is the sum of three counters:

| Counter | Incremented by | Meaning |
|---------|---------------|---------|
| Preemption disable count | `preempt_disable()`, `spin_lock()` variants | Kernel code that must not be preempted |
| Softirq count | `local_bh_disable()`, entering softirq | In softirq context |
| Hardirq count | `irq_enter()` | In hardware interrupt context |

Preemption is allowed only when `preempt_count == 0`. Any spinlock acquisition disables preemption (since sleeping while holding a spinlock would be deadlock-prone).

### TIF_NEED_RESCHED

The `TIF_NEED_RESCHED` flag in `thread_info->flags` signals that the scheduler should be invoked at the next opportunity. It is set by:
- `scheduler_tick()` when the current process's time slice expires
- `try_to_wake_up()` when a higher-priority process is awakened
- `set_tsk_need_resched()` by various kernel paths

The flag is checked at:
- Return from interrupt / exception to kernel mode (if preemption is enabled)
- Return from interrupt / exception to user mode
- After `preempt_enable()` (re-checks the flag)
- Explicit calls to `schedule()` / `cond_resched()`

## Choosing the Right Primitive

| Situation | Recommended primitive |
|-----------|----------------------|
| Data is per-CPU, no cross-CPU sharing needed | Per-CPU variables |
| Single counter or flag | `atomic_t` / atomic bit ops |
| Short critical section, cannot sleep, process context only | `spin_lock()` |
| Short critical section, shared with interrupt handler | `spin_lock_irq()` / `spin_lock_irqsave()` |
| Short critical section, shared with softirq/tasklet | `spin_lock_bh()` |
| Read-mostly data, short critical section | `rwlock_t` or seqlock |
| Read-mostly pointer-based structure, very high read frequency | RCU |
| Simple value read extremely often, written very rarely (e.g., time) | Seqlock |
| Long critical section, may need to sleep | Semaphore (or mutex) |
| Read-heavy, may need to sleep | `rw_semaphore` |
| Wait for one-shot event completion | Completion |
| Legacy code path not yet converted | BKL (avoid in new code) |

## See also

- [interrupt-handling](../entities/interrupt-handling.md)
- [timing-subsystem](../entities/timing-subsystem.md)
- [process-management](../entities/process-management.md)
- [concept-memory-addressing](concept-memory-addressing.md)
