---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [professional-linux-kernel-architecture]
tags: [scheduler, cfs, red-black-tree, vruntime, scheduling-classes]
---

# Completely Fair Scheduler (CFS)

## Overview

The Completely Fair Scheduler (CFS) replaced the O(1) scheduler in Linux 2.6.23, authored by
Ingo Molnar. Rather than relying on heuristic-based priority manipulation to estimate task
interactivity, CFS models an **ideal multitasking CPU** in which every runnable task receives
exactly 1/n of the available processor time, where n is the number of runnable tasks.

The central abstraction is **virtual runtime (vruntime)**: a per-task counter that tracks how
much "ideal" CPU time the task has consumed. At every scheduling decision the kernel picks the
task with the smallest vruntime, ensuring that no task falls unfairly behind. The data structure
backing this selection is a red-black tree, giving O(log n) insertion and O(1) next-task
selection.

CFS is implemented as the `fair_sched_class` within the kernel's
[concept-scheduling-classes](../concepts/concept-scheduling-classes.md) framework. It handles all `SCHED_NORMAL` and
`SCHED_BATCH` tasks (the vast majority of user-space workloads).

## Virtual Runtime (vruntime)

Each task's `vruntime` advances proportionally to wall-clock execution time but inversely
proportionally to the task's **weight** (derived from its nice value). A higher-weight task
(lower nice value) accumulates vruntime more slowly and therefore receives more actual CPU time
before being preempted.

The core formula executed inside `update_curr()` is:

```
vruntime += delta_exec * NICE_0_LOAD / weight
```

Where:

- `delta_exec` is the wall-clock time elapsed since the last accounting update (nanoseconds).
- `NICE_0_LOAD` is the weight of a nice-0 task (1024).
- `weight` is the task's weight from the `prio_to_weight[]` table.

### The Weight Table

The `prio_to_weight[]` array maps each nice level (-20 to +19) to an integer weight. The ratio
between adjacent nice levels is approximately **1.25x** (roughly 10% more or less CPU per nice
step). A few representative values:

| Nice | Weight |
|------|--------|
| -20  | 88761  |
| -10  | 9548   |
|   0  | 1024   |
| +10  | 110    |
| +19  | 15     |

A nice-0 task running alongside a nice-5 task receives proportionally more CPU because its
higher weight causes its vruntime to advance more slowly.

## Red-Black Tree

CFS maintains a per-runqueue red-black tree (`cfs_rq->tasks_timeline`) keyed by vruntime. The
tree provides:

- **O(log n) insertion and deletion** when tasks become runnable or block.
- **O(1) next-task selection** via a cached pointer to the leftmost (minimum vruntime) node
  (`cfs_rq->rb_leftmost`).

When the scheduler needs the next task, it reads `rb_leftmost` directly rather than walking the
tree. Every enqueue or dequeue operation maintains this cached pointer, so selection remains
constant-time regardless of the number of runnable tasks.

## Key Data Structures

### `struct sched_entity`

Embedded inside `task_struct` (field `se`). Represents a schedulable entity, which may be
either a single task or a task group (see Group Scheduling below).

Key fields:

| Field | Purpose |
|-------|---------|
| `vruntime` | Virtual runtime consumed by this entity |
| `load` | Weight of the entity (struct `load_weight`) |
| `on_rq` | Whether the entity is currently on a runqueue |
| `rb_node` | Node in the CFS red-black tree |
| `sum_exec_runtime` | Total wall-clock execution time (nanoseconds) |
| `prev_sum_exec_runtime` | Snapshot of `sum_exec_runtime` at the last scheduling event |

The separation of `sched_entity` from `task_struct` is deliberate: it enables hierarchical
group scheduling, where a single `sched_entity` can front an entire cgroup of tasks.

### `struct cfs_rq`

The per-CPU CFS runqueue. One is embedded in the main `struct rq`, and additional instances
exist for each task group when group scheduling is enabled.

Key fields:

| Field | Purpose |
|-------|---------|
| `load` | Total weight of all entities on this runqueue |
| `nr_running` | Number of runnable entities |
| `min_vruntime` | Monotonically increasing floor; used to place new/woken tasks |
| `tasks_timeline` | Root of the red-black tree |
| `rb_leftmost` | Cached pointer to the leftmost (next-to-run) node |
| `curr` | Pointer to the currently executing `sched_entity` |

`min_vruntime` is critical: it advances as tasks run and is never allowed to go backward. It
serves as the baseline for placing newly created or recently woken tasks so they do not
monopolize the CPU by starting with a very small vruntime.

### `struct rq`

The main per-CPU runqueue. Contains embedded sub-runqueues for each scheduling class:

- `cfs_rq` for CFS (fair class)
- `rt_rq` for real-time tasks
- Pointers to the `current` task, the `idle` task, and scheduling statistics

The [process-scheduler](process-scheduler.md) uses `struct rq` as its central per-CPU data structure.

## Scheduling Classes

CFS is registered as the `fair_sched_class`, an instance of `struct sched_class`. Each
scheduling class provides a table of function pointers:

| Callback | Purpose |
|----------|---------|
| `enqueue_task` | Add a task to the class's runqueue |
| `dequeue_task` | Remove a task from the class's runqueue |
| `check_preempt_curr` | Decide if a newly woken task should preempt the current one |
| `pick_next_task` | Select the next task to run from this class |
| `put_prev_task` | Called when the current task is being switched out |
| `set_curr_task` | Called when a task becomes the current task (e.g., after migration) |
| `task_tick` | Per-tick accounting and preemption check |
| `task_fork` | Set up scheduling state for a newly forked task |

The classes form a strict priority chain via a `next` pointer:

```
rt_sched_class -> fair_sched_class -> idle_sched_class
```

The scheduler iterates this chain from highest to lowest priority: if the real-time class has
runnable tasks, they always run before CFS tasks. The idle class runs only when no other class
has work. See [concept-scheduling-classes](../concepts/concept-scheduling-classes.md) for a full discussion of the framework.

## Key Operations

### `update_curr()`

The core accounting function. Called on every timer tick, on task wakeup, and at various other
scheduling events. It computes `delta_exec` (time since last update), adds the weighted delta
to the current entity's `vruntime`, and advances `min_vruntime`.

### `pick_next_task_fair()`

Returns the `sched_entity` at the leftmost position in the red-black tree. If the tree is
empty, returns NULL so the scheduler moves to the next class in the priority chain.

### `enqueue_task_fair()` / `dequeue_task_fair()`

Insert or remove a `sched_entity` from the rb-tree. These functions also update the runqueue's
aggregate load (`cfs_rq->load`) and `nr_running` count, and maintain the `rb_leftmost` cache.

### `task_tick_fair()`

Called on every timer interrupt. Invokes `update_curr()` to refresh the current task's vruntime,
then checks whether the current task's vruntime has exceeded the leftmost entity's vruntime by
at least `sysctl_sched_wakeup_granularity`. If so, it sets the `TIF_NEED_RESCHED` flag to
trigger preemption at the next safe point.

### `place_entity()`

Sets the initial vruntime for a newly created or recently woken task. Uses `min_vruntime` as a
baseline to prevent starvation: a new task cannot start with a vruntime far below the current
minimum, which would let it monopolize the CPU.

## Sleeper Fairness

When a task wakes up after sleeping, its vruntime may be far behind `min_vruntime`. CFS applies
a bounded credit:

```
vruntime = max(old_vruntime, min_vruntime - sched_latency / 2)
```

This places the woken task slightly to the left of currently runnable tasks in the rb-tree,
giving it a **small scheduling bonus** that lets it run soon after waking. The bonus is capped
at half the scheduling latency period, preventing long sleepers from monopolizing the CPU for
an extended burst upon waking.

This mechanism replaces the O(1) scheduler's fragile `sleep_avg` heuristic with a bounded,
predictable policy.

## Granularity Controls

CFS exposes several tunables under `/proc/sys/kernel/`:

| Tunable | Default | Purpose |
|---------|---------|---------|
| `sched_latency_ns` | ~20 ms | Target preemption latency: the period over which all runnable tasks should each run at least once |
| `sched_min_granularity_ns` | ~4 ms | Minimum time slice a task receives before it can be preempted |
| `sched_wakeup_granularity_ns` | ~4 ms | Minimum vruntime advantage a waking task must have over the current task to trigger preemption |

When the number of runnable tasks grows large, the effective time slice per task
(`sched_latency / nr_running`) can drop below `sched_min_granularity`. In that case CFS clamps
the slice to `sched_min_granularity`, and the actual scheduling period stretches beyond
`sched_latency`.

These tunables let administrators trade off between interactive responsiveness (lower latency)
and throughput (larger granularity, fewer context switches).

## Group Scheduling

With `CONFIG_FAIR_GROUP_SCHED` enabled, CFS supports hierarchical scheduling through cgroups.
A `sched_entity` can represent an entire task group rather than a single task. Each group
receives its own `cfs_rq`.

The hierarchy works as follows:

1. The **top-level** `cfs_rq` (on each CPU) schedules among group-level `sched_entity`
   instances and ungrouped tasks.
2. When a group entity is selected, the scheduler descends into that group's `cfs_rq` and
   picks the next task within the group.
3. This nesting can be multiple levels deep, following the cgroup hierarchy.

Group scheduling ensures that CPU bandwidth is divided fairly among groups first, then among
tasks within each group. For example, if group A has 10 tasks and group B has 1 task, each
group gets 50% of the CPU (assuming equal group weights), and group A's 10 tasks each get 5%
of total CPU time.

## Comparison with the O(1) Scheduler

| Aspect | O(1) Scheduler | CFS |
|--------|---------------|-----|
| Interactivity detection | Heuristic `sleep_avg` (could be gamed) | Mathematically fair vruntime model |
| Data structure | Two priority arrays with bitmap | Red-black tree keyed by vruntime |
| Next-task selection | O(1) via bitmap scan | O(1) via cached `rb_leftmost` pointer |
| Task insertion | O(1) into priority array | O(log n) into rb-tree |
| Fairness guarantee | Approximate (heuristic-dependent) | Proportional to weight (provable) |
| Time slice calculation | Fixed slices per priority level | Dynamic, derived from weight ratios and `sched_latency` |

CFS trades a constant-time insertion for a stronger fairness model and simpler, more
maintainable code. In practice the O(log n) insertion cost is negligible because the number of
runnable tasks on a single CPU is typically small (tens to low hundreds).

## See also

- [process-scheduler](process-scheduler.md)
- [process-management](process-management.md)
- [concept-scheduling-classes](../concepts/concept-scheduling-classes.md)
- [system-calls](system-calls.md)
