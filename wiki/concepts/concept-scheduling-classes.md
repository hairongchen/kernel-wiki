---
type: concept
created: 2026-04-09
updated: 2026-04-09
sources: [professional-linux-kernel-architecture]
tags: [scheduler, scheduling-classes, cfs, real-time, design-pattern]
---

# Scheduling Classes

## Overview

Scheduling classes are a pluggable scheduling framework introduced alongside CFS in Linux 2.6.23. They separate scheduling **policy** from **mechanism**, allowing multiple scheduling algorithms to coexist within the same kernel. Rather than a monolithic scheduler that uses conditional branches to handle different task types, each scheduling policy is encapsulated in its own self-contained class.

This is an instance of the "pluggable policies" design pattern pervasive throughout the Linux kernel. The core scheduler provides the framework — timekeeping, context switching, CPU selection — while each scheduling class implements the policy-specific logic for task ordering, preemption decisions, and fairness accounting.

## `struct sched_class`

The interface that every scheduling class must implement is defined by `struct sched_class`. It is a vtable of function pointers, each corresponding to a specific scheduler operation:

| Function pointer | Signature | Purpose |
|---|---|---|
| `enqueue_task` | `(rq, task, flags)` | Add a task to the class's runqueue. Called when a task becomes runnable. |
| `dequeue_task` | `(rq, task, flags)` | Remove a task from the class's runqueue. Called when a task blocks or exits. |
| `check_preempt_curr` | `(rq, task, flags)` | Determine whether a newly woken or newly arrived task should preempt the currently running task. |
| `pick_next_task` | `(rq)` | Select the next task to run from this class's runqueue. Returns NULL if no tasks are runnable. |
| `put_prev_task` | `(rq, task)` | Called when a task is being switched away from. Allows the class to update accounting. |
| `set_curr_task` | `(rq)` | Called when a task changes its scheduling class (e.g., via `sched_setscheduler()`). |
| `task_tick` | `(rq, task, queued)` | Periodic timer callback invoked on each scheduler tick. Used for timeslice accounting and preemption checks. |
| `task_fork` | `(task)` | Called when a task forks a new child. Allows the class to initialize scheduling state for the child. |
| `yield_task` | `(rq)` | Handle an explicit `sched_yield()` call from userspace. |
| `select_task_rq` | `(task, sd_flag, flags)` | Select the most appropriate CPU for task placement. Used on SMP systems for load balancing and affinity decisions. |

Not every function pointer is mandatory for every class. For instance, `idle_sched_class` has minimal implementations for most operations since it only ever runs the per-CPU idle task.

## Class Hierarchy

Scheduling classes are linked together by a `next` pointer, forming a strict priority chain. The chain is traversed from highest to lowest priority:

```
stop_sched_class → rt_sched_class → fair_sched_class → idle_sched_class
```

When the scheduler needs to select the next task to run, it walks this chain. The first class that has a runnable task wins. This means any runnable stop-class task will always preempt any RT task, any RT task will always preempt any CFS task, and so on.

### stop_sched_class

The highest-priority class. Used exclusively for kernel-internal migration and stopper threads. Tasks in this class cannot be preempted by anything. This class exists to support operations that must run on a specific CPU without interruption, such as CPU hotplug transitions and task migration.

### rt_sched_class

Handles `SCHED_FIFO` and `SCHED_RR` real-time tasks. Uses a bitmap-based priority scheme with 100 priority levels (0-99). Task selection is O(1): find the highest-priority bit set in the bitmap, then take the first task from the corresponding per-priority FIFO queue. `SCHED_FIFO` tasks run until they voluntarily yield or block; `SCHED_RR` tasks additionally have a time quantum after which they rotate to the end of their priority queue.

### fair_sched_class

The default class for ordinary tasks (`SCHED_NORMAL`, `SCHED_BATCH`). Implements [Completely Fair Scheduling (CFS)](../entities/cfs-scheduler.md).md) using a red-black tree keyed on virtual runtime (`vruntime`). The leftmost node in the tree (the task with the smallest `vruntime`) is always the next to run. This provides O(log n) insertion and O(1) selection of the next task (the leftmost node is cached).

### idle_sched_class

The lowest-priority class. Manages the per-CPU idle task, which runs only when no other class has any runnable task. The idle task typically executes architecture-specific power-saving instructions (e.g., `hlt` on x86).

## Dispatch Mechanism

The main scheduler entry point `schedule()` calls `pick_next_task(rq)`, which iterates through the class chain by following the `next` pointers:

```
for_each_class(class) {
    p = class->pick_next_task(rq);
    if (p)
        return p;
}
```

An important optimization exists for the common case: if all runnable tasks on the runqueue belong to the fair class (checked by comparing `rq->nr_running == rq->cfs_rq.nr_running`), the scheduler skips the chain walk entirely and goes directly to CFS. On a typical desktop or server workload, this shortcut fires nearly every time, avoiding the overhead of checking the stop and RT classes.

## Per-Class Runqueue State

Each scheduling class maintains its own runqueue data structure, embedded within the per-CPU `struct rq`:

- **`struct cfs_rq`** (fair class): Contains the red-black tree of tasks keyed by `vruntime`, the `min_vruntime` watermark for the runqueue, per-entity load tracking data, and group scheduling hierarchy information. See [cfs-scheduler](../entities/cfs-scheduler.md) for details.

- **`struct rt_rq`** (RT class): Contains a priority bitmap (`rt_prio_bitmap`) and an array of linked lists, one per priority level. The bitmap allows O(1) lookup of the highest-priority non-empty queue using `sched_find_first_bit()`.

- **idle class**: Has no dedicated runqueue structure. The per-CPU idle task is referenced directly from `struct rq` and is always available as a fallback.

This separation ensures that each class's internal bookkeeping does not interfere with other classes and can evolve independently.

## Design Pattern: Vtable-Based Dispatch

The `struct sched_class` pattern is a recurring motif throughout the Linux kernel. It is structurally identical to:

- **`struct file_operations`** in VFS — each filesystem implements open, read, write, etc.
- **`struct proto`** in networking — each protocol (TCP, UDP, SCTP) provides its own socket operations.
- **`struct irq_chip`** in genirq — each interrupt controller implements ack, mask, unmask, etc.

In each case, a vtable of function pointers is paired with a registration/selection mechanism, enabling the kernel to support multiple implementations behind a uniform interface. For scheduling classes, the selection mechanism is the priority chain walk at dispatch time.

This design made it straightforward to add `SCHED_DEADLINE` (deadline scheduling class) in a later kernel release — it was inserted into the priority chain between `rt_sched_class` and `fair_sched_class` without modifying any existing class.

## Comparison with the O(1) Scheduler

The O(1) scheduler (Linux 2.6.0 through 2.6.22) used a single monolithic implementation with `if`/`else` branches to distinguish RT tasks from conventional tasks. Policy-specific logic was intermixed with the core scheduling framework, making it difficult to maintain and extend.

The `sched_class` abstraction cleanly decomposes this: each class is a self-contained module with its own data structures and algorithms. The core scheduler is reduced to a thin dispatch layer. This separation improved both maintainability (changes to CFS cannot break RT scheduling) and extensibility (new classes can be added without touching existing code).

## See also

- [cfs-scheduler](../entities/cfs-scheduler.md)
- [process-scheduler](../entities/process-scheduler.md)
- [process-management](../entities/process-management.md)
