---
type: entity
created: 2026-04-08
updated: 2026-04-09
sources: [understanding-the-linux-kernel, professional-linux-kernel-architecture]
tags: [scheduler, scheduling, smp, load-balancing, cfs]
---

# Process Scheduler

## O(1) Scheduler (Linux 2.6.0 -- 2.6.22)

The Linux 2.6 process scheduler, introduced by Ingo Molnar, achieves O(1) time complexity for all core scheduling operations — selecting the next task, inserting a task, and removing a task. This replaced the O(n) scheduler of Linux 2.4, which degraded significantly as the number of runnable processes grew.

### Design Overview

The scheduler's key design goals are:

1. **O(1) complexity** for all scheduling decisions regardless of system load.
2. **Perfect SMP scalability** via per-CPU runqueues with no global locks.
3. **Good interactive response** by detecting and favoring I/O-bound (interactive) processes.
4. **SMP affinity** to keep processes on the same CPU for cache warmth, balanced against fairness.

## Priority Model

The scheduler manages 140 priority levels, divided into two classes:

| Range | Class | Description |
|-------|-------|-------------|
| 0 -- 99 | Real-time | Used by `SCHED_FIFO` and `SCHED_RR` tasks. Lower number = higher priority. These always preempt conventional tasks. |
| 100 -- 139 | Conventional | Used by `SCHED_NORMAL` tasks. Maps to nice values -20 (priority 100) through +19 (priority 139). |

### Static Priority

Every conventional process has a **static priority** set by the `nice()` system call. The static priority determines:

- The base time quantum.
- The starting point for dynamic priority calculation.

The default static priority is 120 (nice 0).

### Dynamic Priority

The **dynamic priority** (`prio` field in `task_struct`) is what the scheduler actually uses to make decisions. For conventional processes, it is computed as:

```
dynamic priority = max(100, min(static_priority - bonus + 5, 139))
```

The **bonus** ranges from 0 to 10 and is derived from the process's `sleep_avg` field, which tracks how much time the process spends sleeping versus running:

- A process that sleeps a lot (I/O-bound / interactive) accumulates a high `sleep_avg` and receives a bonus up to +5 (lower numerical priority = higher actual priority).
- A process that runs continuously (CPU-bound) has its `sleep_avg` drain, receiving a penalty of up to -5 (higher numerical priority = lower actual priority).

This mechanism automatically distinguishes interactive from batch workloads without explicit user configuration.

## Time Quantum

Each process receives a **time quantum** (time slice) that is proportional to its static priority:

| Static Priority (nice) | Time Quantum |
|------------------------|--------------|
| 100 (nice -20) | 800 ms |
| 120 (nice 0) | 100 ms (default) |
| 139 (nice +19) | 5 ms |

The formula for the base time quantum is:

```
if (static_priority < 120)
    quantum = (140 - static_priority) * 20  ms
else
    quantum = (140 - static_priority) * 5   ms
```

When a process exhausts its time quantum, it is moved from the active array to the expired array (unless it qualifies as interactive — see below).

## Priority Arrays and Bitmap Scheduling

The core of the O(1) scheduler is the **priority array** data structure (`struct prio_array`), which contains:

1. **An array of 140 linked lists** — one list per priority level, holding all runnable tasks at that priority.
2. **A bitmap of 140 bits** — bit `i` is set if priority level `i` has at least one runnable task.
3. **A count** of the total number of tasks in the array (`nr_active`).

### Selecting the Next Task

Finding the highest-priority runnable task is O(1):

1. Use `sched_find_first_bit()` on the bitmap to find the first set bit. On x86, this compiles to a `bsfl` (bit-scan-forward) instruction — constant time.
2. Pick the first task from the corresponding linked list.

No iteration over all tasks is ever needed.

### Active and Expired Arrays

Each per-CPU runqueue maintains two priority arrays:

- **Active array**: Contains tasks that still have time quantum remaining.
- **Expired array**: Contains tasks that have exhausted their time quantum.

When a task's quantum expires, `scheduler_tick()` moves it from the active to the expired array and recalculates its dynamic priority and time quantum. This recalculation is distributed — each process's priority is recalculated individually when it exhausts its slice, avoiding the global "epoch recalculation" of the O(n) scheduler.

When the active array becomes empty (all tasks have exhausted their quanta), the scheduler simply **swaps the active and expired array pointers** — an O(1) operation. The formerly expired array, now full of tasks with fresh time quanta, becomes the new active array.

```
Before swap:
  active  -> [empty arrays]
  expired -> [tasks with fresh quanta]

After swap (pointer exchange):
  active  -> [tasks with fresh quanta]
  expired -> [empty arrays]
```

### Interactive Process Reinsertion

To maintain interactive responsiveness, the scheduler applies a heuristic: if a task that exhausts its time quantum is deemed **interactive** (its dynamic priority is significantly boosted by a high `sleep_avg`), it is reinserted into the **active** array rather than the expired array. This prevents interactive processes (text editors, shells, GUI applications) from experiencing latency spikes when the active/expired swap occurs.

The threshold is: a process is considered interactive if its bonus is large enough that its dynamic priority is at least 3 levels above the "interactive delta" relative to its static priority.

A safeguard mechanism (`expired_timestamp`) ensures that if the expired array has tasks that have waited too long, even interactive tasks are eventually forced to the expired array to prevent starvation.

## Scheduling Policies

### SCHED_NORMAL (SCHED_OTHER)

The default policy for conventional processes. Uses dynamic priorities, time quanta, and the active/expired array mechanism described above.

### SCHED_FIFO (First In, First Out Real-Time)

- Always has priority over `SCHED_NORMAL` tasks (priorities 0--99).
- A `SCHED_FIFO` task runs until it voluntarily yields, blocks, or is preempted by a higher-priority real-time task.
- It has **no time quantum** — it will not be preempted due to time expiration.
- Among tasks of the same SCHED_FIFO priority, the one that arrived first runs first.

### SCHED_RR (Round-Robin Real-Time)

- Identical to `SCHED_FIFO` except that tasks at the same priority level are round-robin scheduled — each gets a time quantum, and when it expires, the task moves to the back of the queue for its priority level.
- Higher-priority real-time tasks still preempt unconditionally.

## Per-CPU Runqueues

Each CPU has its own `struct runqueue` (later renamed `struct rq`), which contains:

| Field | Purpose |
|-------|---------|
| `lock` | Spinlock protecting the entire runqueue. Per-CPU locks eliminate global contention. |
| `nr_running` | Number of runnable tasks on this CPU. |
| `cpu_load` | Exponentially weighted moving average of the CPU's load. |
| `active` | Pointer to the active priority array. |
| `expired` | Pointer to the expired priority array. |
| `arrays[2]` | The two actual priority array structures (active and expired are pointers into these). |
| `expired_timestamp` | When the first task was inserted into the expired array. Used for starvation prevention. |
| `curr` | Pointer to the currently running task. |
| `idle` | Pointer to the idle task for this CPU. |
| `migration_thread` | The migration kernel thread for this CPU. |

Because each CPU has its own runqueue and its own lock, the scheduler scales linearly with the number of CPUs — no CPU needs to acquire another CPU's lock during normal scheduling.

## Key Functions

### schedule()

The main scheduler function, called when:
- A process voluntarily sleeps (calls `schedule()` directly).
- A process returns from a system call or interrupt with `TIF_NEED_RESCHED` set.
- A process's time quantum expires.

`schedule()` performs:
1. Disables preemption.
2. Examines the current task: if it is not `TASK_RUNNING`, removes it from the runqueue (it is sleeping or dying).
3. Checks if the active array is empty; if so, swaps active and expired pointers.
4. Calls `sched_find_first_bit()` on the active bitmap to find the highest-priority task.
5. If the selected task differs from the current task, calls `context_switch()` to perform the actual [context switch](process-management.md).
6. Re-enables preemption.

### scheduler_tick()

Called on every timer interrupt (typically every 1 ms with `HZ=1000`). It:

1. Updates time-accounting fields (`sched_clock`).
2. Decrements the current task's remaining time quantum.
3. If the quantum reaches zero:
   - Recalculates the task's dynamic priority.
   - Assigns a new time quantum.
   - Moves the task to the expired array (or reinserts to active if interactive).
   - Sets `TIF_NEED_RESCHED` to trigger `schedule()` on return to user space.
4. Periodically triggers `rebalance_tick()` for SMP load balancing.

### try_to_wake_up()

Wakes up a sleeping process by:
1. Recalculating the process's dynamic priority (it may have accumulated sleep bonus).
2. Setting the task to `TASK_RUNNING`.
3. Selecting an appropriate CPU (possibly different from the one the task last ran on, if that CPU is overloaded).
4. Inserting the task into the chosen CPU's active array.
5. Setting `TIF_NEED_RESCHED` on the chosen CPU's current task if the newly woken task has higher priority.

### context_switch()

Performs the actual CPU context switch. Delegates to `switch_mm()` for the address space switch and `switch_to()` for the register state switch. See [process-management](process-management.md) for details.

## SMP Load Balancing

On multiprocessor systems, the scheduler must distribute work across CPUs to prevent imbalances where some CPUs are idle while others are overloaded.

### Scheduling Domains

Linux 2.6 models the CPU topology as a hierarchy of **scheduling domains** (`struct sched_domain`), where each domain represents a level of hardware sharing:

| Level | Example | Balancing Frequency | Cost of Migration |
|-------|---------|--------------------|--------------------|
| SMT domain | Hyperthreaded siblings | Very frequent (every 1-2 ms) | Very low (shared L1 cache) |
| SMP domain | Physical CPUs in same package/node | Moderate (every 4-8 ms) | Moderate (shared L2/L3) |
| NUMA domain | CPUs in different NUMA nodes | Infrequent (every 16-64 ms) | High (cross-node memory access) |

Each scheduling domain contains one or more **scheduling groups** (`struct sched_group`), where each group corresponds to a set of CPUs at the same level. The domain/group hierarchy allows the balancer to make topology-aware decisions.

### load_balance()

The core balancing function, called periodically by `rebalance_tick()` and also when a CPU goes idle. For each scheduling domain, starting from the lowest level:

1. **Find the busiest group** in the domain by comparing load averages (`cpu_load`).
2. **Find the busiest CPU** in that group.
3. **Pull tasks** from the busiest CPU's runqueue to the current CPU's runqueue.
4. Prefer to pull tasks that are:
   - Not currently running.
   - Not cache-hot (recently ran on the source CPU — migration would cause cache misses).
   - Not pinned to the source CPU via CPU affinity (`cpus_allowed`).

If no imbalance is found or tasks cannot be moved (e.g., all are cache-hot or affinity-pinned), the balancer backs off and retries later.

### Migration Threads

Each CPU has a kernel thread named `migration/N` running at real-time priority 99. These threads handle explicit migration requests:

- When a process's CPU affinity is changed and it is currently on a CPU it is no longer allowed to use.
- When a CPU is being taken offline (hotplug).
- When `load_balance()` determines that a task should be moved but cannot do so from the current context.

The migration thread on the source CPU dequeues the task and enqueues it on the destination CPU's runqueue.

### Idle Balancing

When a CPU has no runnable tasks and enters the idle loop, it immediately calls `load_balance()` in idle mode, using more aggressive thresholds to pull work from busy CPUs. This ensures idle CPUs absorb work quickly.

## Kernel Preemption

Linux 2.6 supports **kernel preemption**: a process running in kernel mode can be preempted (switched out) if it is not holding any locks. The `preempt_count` field in `thread_info` tracks whether preemption is safe:

- `preempt_count == 0` and `TIF_NEED_RESCHED` is set: preemption can occur.
- `preempt_count > 0`: the task holds a spinlock or is in an atomic section; preemption is deferred.

Preemption points exist at `spin_unlock()`, `preempt_enable()`, and return from interrupt to kernel mode.

## CFS: Completely Fair Scheduler (Linux 2.6.23+)

The O(1) scheduler was replaced by the **Completely Fair Scheduler (CFS)** in Linux 2.6.23, designed by Ingo Molnar. CFS abandons the heuristic-based priority arrays in favor of a mathematically fair model.

### Core Idea

CFS models an "ideal multitasking CPU" where each of `n` tasks receives exactly `1/n` of CPU time. It tracks each task's **virtual runtime** (`vruntime`) — the amount of weighted CPU time consumed. Tasks with lower `vruntime` (i.e., those that have received less than their fair share) are scheduled next.

The key formula: `vruntime += delta_exec * NICE_0_LOAD / weight`, where `weight` is derived from the nice value via `prio_to_weight[]`. Higher-weight tasks (lower nice) accumulate `vruntime` more slowly, receiving proportionally more CPU time.

### Data Structures

- **`struct sched_entity`** — per-task scheduling entity embedded in `task_struct`. Holds `vruntime`, load weight, and an rb-tree node. Also supports group scheduling.
- **`struct cfs_rq`** — per-CPU CFS runqueue. Contains a red-black tree (`tasks_timeline`) sorted by `vruntime`, with the leftmost node cached for O(1) next-task selection.

### Scheduling Classes

CFS introduced the **scheduling class** abstraction (`struct sched_class`), enabling pluggable scheduling policies. Classes form a priority chain: `rt_sched_class` → `fair_sched_class` → `idle_sched_class`. The scheduler iterates this chain, selecting the first class with runnable tasks. See [concept-scheduling-classes](../concepts/concept-scheduling-classes.md) for details.

For full CFS coverage, see [cfs-scheduler](cfs-scheduler.md).

## See also

- [process-management](process-management.md)
- [system-calls](system-calls.md)
- [signals](signals.md)
- [interrupt-handling](interrupt-handling.md)
- [cfs-scheduler](cfs-scheduler.md)
- [concept-scheduling-classes](../concepts/concept-scheduling-classes.md)
