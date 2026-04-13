---
type: entity
created: 2026-04-08
updated: 2026-04-09
sources: [understanding-the-linux-kernel, advanced-linux-programming]
tags: [process-management, task-struct, scheduling, context-switch, fork, exec]
---

# Process Management

Process management is the core subsystem responsible for creating, scheduling, and destroying the units of execution in the Linux kernel. In Linux, there is no fundamental distinction between a "process" and a "thread" at the kernel level — both are represented by the same data structure (`task_struct`) and differ only in which resources they share.

## The Process Descriptor: task_struct

Every process (or thread) in the system is represented by a `task_struct`, defined in `<linux/sched.h>`. This structure is approximately 1.7 KB in size and contains all the information the kernel needs to manage a process:

| Field(s) | Purpose |
|----------|---------|
| `state` | Current process state (running, sleeping, zombie, etc.) |
| `pid` | Process ID — unique numeric identifier for this task |
| `tgid` | Thread Group ID — the PID of the thread group leader; this is what userspace sees as the "PID" via `getpid()` |
| `thread_info` | Pointer to the low-level thread information stored at the base of the kernel stack |
| `mm` / `active_mm` | Pointer to the `mm_struct` describing the process's virtual address space |
| `files` | Pointer to `files_struct` — the open file descriptor table |
| `fs` | Pointer to `fs_struct` — root and current working directory |
| `signal` | Pointer to `signal_struct` — shared signal state for the thread group |
| `sighand` | Pointer to `sighand_struct` — signal handler table |
| `parent`, `children`, `sibling` | Process hierarchy links |
| `policy`, `prio`, `static_prio` | [Scheduling](process-scheduler.md) policy and priority fields |
| `uid`, `euid`, `gid`, `egid` | Process credentials |
| `comm[TASK_COMM_LEN]` | Executable name (for debugging / /proc display) |
| `thread` | Architecture-specific `thread_struct` holding saved CPU register state |

The `task_struct` instances for all processes are linked in a circular doubly-linked list (via `tasks` field) and also organized into a hash table keyed by PID for fast lookup.

## thread_info and the Kernel Stack

Each process has an 8 KB (two-page) kernel-mode stack. At the very base (lowest address) of this stack sits the `thread_info` structure, which contains:

- A pointer back to the full `task_struct`
- CPU-specific flags (e.g., `TIF_NEED_RESCHED`, `TIF_SIGPENDING`)
- The preemption counter (`preempt_count`)
- Supervisor stack pointer

The `current` macro retrieves the `task_struct` of the currently running process. On x86, it works by masking the lower 13 bits of the kernel stack pointer to find the `thread_info` at the stack base, then following the pointer to `task_struct`. This is an O(1) operation that requires no global variable lookup.

```
+--------------------+  <-- high address (top of 8 KB region)
|                    |
|   Kernel stack     |
|   (grows down)     |
|                    |
+--------------------+
|   thread_info      |  <-- low address (base of 8 KB region)
+--------------------+
```

## Process States

The `state` field of `task_struct` holds one of the following values:

| State | Value | Description |
|-------|-------|-------------|
| `TASK_RUNNING` | 0 | The process is either currently executing on a CPU or sitting in a runqueue ready to execute. This is the only state for processes that are "runnable." |
| `TASK_INTERRUPTIBLE` | 1 | Sleeping, waiting for some condition. Can be woken by a signal or by the waited-for event. Most voluntary sleeps use this state. |
| `TASK_UNINTERRUPTIBLE` | 2 | Sleeping and cannot be woken by a signal — only by the specific event it is waiting for. Used when a process must not be interrupted (e.g., during certain disk I/O operations). Processes in this state contribute to the load average. |
| `TASK_STOPPED` | 4 | Execution has been stopped, typically by receiving `SIGSTOP`, `SIGTSTP`, `SIGTTIN`, or `SIGTTOU`. Resumed by `SIGCONT`. Used for job control. |
| `TASK_TRACED` | 8 | Stopped by a debugger via `ptrace()`. Distinct from `TASK_STOPPED` so that ptrace stops are not confused with job-control stops. |
| `EXIT_ZOMBIE` | 16 | The process has terminated but its parent has not yet called `wait()` to collect the exit status. The `task_struct` persists so the parent can read the exit code. |
| `EXIT_DEAD` | 32 | Final state after the parent has collected the status (or the child was auto-reaped). The `task_struct` is about to be freed. |

State transitions follow a well-defined pattern: a new process starts as `TASK_RUNNING`, may sleep (`TASK_INTERRUPTIBLE` or `TASK_UNINTERRUPTIBLE`), and eventually transitions through `EXIT_ZOMBIE` to `EXIT_DEAD` upon termination.

## Process Creation: fork, clone, and do_fork

Linux uses a **clone-based** model for process creation. All of `fork()`, `vfork()`, and `pthread_create()` ultimately invoke the same kernel function: `do_fork()`, differing only in which `clone_flags` they pass.

### The clone() System Call

The `clone()` system call is the most general interface. It accepts a set of flags that control which resources are shared between parent and child:

| Flag | Effect |
|------|--------|
| `CLONE_VM` | Share the memory address space (`mm_struct`). Without this flag, the child gets a copy-on-write duplicate. |
| `CLONE_FILES` | Share the file descriptor table. Without it, the child gets a copy. |
| `CLONE_FS` | Share filesystem information (root dir, cwd, umask). |
| `CLONE_SIGHAND` | Share signal handlers. Requires `CLONE_VM`. |
| `CLONE_THREAD` | Place the new task in the same thread group (same `tgid`). Requires `CLONE_SIGHAND`. |
| `CLONE_PARENT` | The new task has the same parent as the caller (sibling, not child). |
| `CLONE_VFORK` | The parent is suspended until the child calls `exec()` or `_exit()`. Used by `vfork()`. |
| `CLONE_NEWNS` | Give the child its own mount namespace. |
| `CLONE_PTRACE` | If the parent is being traced, also trace the child. |

### do_fork() and copy_process()

`do_fork()` is the core kernel function. Its flow:

1. Calls `copy_process()` to create the new `task_struct`:
   - `dup_task_struct()` — allocates a new `task_struct` and kernel stack, copies the parent's descriptor.
   - Checks resource limits (e.g., `RLIMIT_NPROC`).
   - Initializes various fields (e.g., zeroes out statistics, sets `state` to `TASK_UNINTERRUPTIBLE`).
   - `copy_flags()` — updates process flags (clears `PF_SUPERPRIV`, sets `PF_FORKNOEXEC`).
   - `alloc_pid()` — assigns a new PID.
   - Depending on clone flags, either copies or shares: the open file table (`copy_files`), filesystem info (`copy_fs`), signal handlers (`copy_sighand`), signal state (`copy_signal`), memory descriptor (`copy_mm`), and namespaces (`copy_namespace`).
   - `copy_thread()` — sets up the kernel stack for the child, including the return value of 0 from the "fork" in the child.
2. If `CLONE_THREAD`, the child joins the parent's thread group (same `tgid`).
3. Sets the child state to `TASK_RUNNING` and inserts it into the scheduler's runqueue.
4. If `CLONE_VFORK`, the parent blocks on a completion variable until the child exits or execs.
5. Returns the child's PID to the parent.

### Copy-on-Write (COW)

When `CLONE_VM` is not set (e.g., a traditional `fork()`), the child does not immediately receive a full copy of the parent's address space. Instead, the kernel marks all writable pages in both parent and child as read-only and sets up page-fault handlers to duplicate pages on demand when either process attempts a write. This makes `fork()` extremely fast — especially when followed immediately by `exec()`, which discards the address space entirely.

## Lightweight Processes and Threads

Linux implements POSIX threads as **lightweight processes** (LWPs). A Linux "thread" is simply a `task_struct` that was created with `CLONE_VM | CLONE_FILES | CLONE_FS | CLONE_SIGHAND | CLONE_THREAD`. This means:

- All threads in a group share the same `mm_struct` (virtual memory).
- All threads share the same file descriptor table.
- All threads share the same signal handler table.
- All threads have the same `tgid` (which is reported as the "PID" by `getpid()`), but each has a unique `pid` (visible via `gettid()`).

The thread group leader is the first thread created (its `pid == tgid`). The `signal_struct` is shared across the entire thread group, allowing [signals](signals.md) to be delivered to the group as a whole.

The NPTL (Native POSIX Thread Library) in glibc uses this 1:1 threading model, where each POSIX thread maps to exactly one kernel lightweight process.

## Context Switching

A context switch occurs when the [process-scheduler](process-scheduler.md) decides to run a different process on the current CPU. The key function is `context_switch()`, which performs two operations:

### 1. Address Space Switch

`switch_mm()` updates the page tables by loading the new process's page directory into the `cr3` register (on x86). If the new process is a kernel thread (which has no `mm_struct` of its own), the kernel uses **lazy TLB** mode — it borrows the previous process's page tables via `active_mm` and avoids the expensive TLB flush, since kernel threads only access kernel address space.

### 2. Processor State Switch

The `switch_to` macro (which calls `__switch_to()` on x86) saves and restores the CPU register state:

- Saves the old process's kernel stack pointer (`esp`) and instruction pointer (`eip`) into its `thread_struct`.
- Loads the new process's `esp` and `eip`.
- Updates the TSS (Task State Segment) with the new process's kernel stack pointer (`esp0`), so that subsequent transitions from user mode to kernel mode use the correct stack.
- Switches debug registers if needed.
- Handles I/O permission bitmap updates.

### Lazy FPU Switching

The FPU/MMX/SSE register state is large and expensive to save/restore. Linux uses **lazy FPU switching**: when switching away from a process, the kernel sets the `TS` (Task Switched) flag in the `cr0` register. If the next process attempts to use FPU instructions, a "Device Not Available" exception (#NM) fires, and only then does the kernel save the old FPU state and restore the new process's state. If a process never uses the FPU (common for many system tasks), the save/restore is skipped entirely.

## Process Destruction

When a process terminates (via `exit()`, receiving a fatal signal, etc.), the kernel calls `do_exit()`:

1. Sets `PF_EXITING` flag in `task_struct->flags`.
2. Releases resources: removes timers, releases the `mm_struct` (`exit_mm`), detaches from the file descriptor table (`exit_files`), filesystem info (`exit_fs`), semaphores (`exit_sem`), and namespaces (`exit_namespace`).
3. Sets `exit_code` in the `task_struct`.
4. Calls `exit_notify()`:
   - If the process has children, re-parents them to the init process (PID 1) or to another process in the same thread group.
   - If the process is a thread group leader and all other threads have exited, sends `SIGCHLD` to the parent.
   - Sets the state to `EXIT_ZOMBIE`.
5. Calls `schedule()` to switch to another process. The zombie task never runs again.

The `task_struct` and kernel stack persist in the zombie state until the parent calls `wait()` / `waitpid()`, which invokes `release_task()` to:

- Decrement the process count for the user.
- Remove the process from the PID hash table and the task list.
- Free the `task_struct` and `thread_info`/stack memory.

If a parent exits without waiting on its children, init (PID 1) inherits and automatically reaps them, preventing permanent zombies.

## Kernel Threads

Kernel threads are processes that run exclusively in kernel mode — they have no user-space address space (`mm_struct` is `NULL`). They are used for background tasks such as flushing dirty pages, handling softirqs, and CPU migration.

### Notable Kernel Threads

| Thread | PID | Purpose |
|--------|-----|---------|
| **swapper** (idle task) | 0 | One per CPU. Created at boot. Runs the idle loop when no other process is runnable. Each CPU's process 0 is the ancestor of all processes on that CPU. |
| **init** | 1 | The first user-space process. Ancestor of all user-space processes. Responsible for reaping orphaned zombies. |
| **ksoftirqd/N** | — | Per-CPU thread for processing software interrupts under heavy load. |
| **kswapd** | — | Page reclamation daemon for memory management. |
| **pdflush** | — | Flushes dirty pages from the page cache back to disk. |
| **migration/N** | — | Per-CPU, priority-99 threads used by the scheduler to move tasks between CPUs. |

Kernel threads are created via `kernel_thread()`, which calls `do_fork()` with the `CLONE_VM` flag (they share the kernel's address space) and `CLONE_UNTRACED` to prevent ptrace attachment. The child function runs in kernel mode and typically enters an infinite loop, sleeping until work arrives.

## Wait Queues

Wait queues are the primary mechanism for processes to sleep until a condition is met. A wait queue is a linked list of `wait_queue_t` entries, each referencing a sleeping task and a wake-up function.

### Exclusive vs. Non-exclusive Wakeups

When an event occurs and processes in a wait queue are woken:

- **Non-exclusive waiters** (`WQ_FLAG_EXCLUSIVE` not set): All non-exclusive waiters are woken up. This is the default. Suitable when all waiters need to respond to the event.
- **Exclusive waiters** (`WQ_FLAG_EXCLUSIVE` set): Only one exclusive waiter is woken (in addition to all non-exclusive waiters). This prevents the **thundering herd** problem, where many processes wake up but only one can actually make progress (e.g., accepting a connection on a socket). Exclusive waiters are added to the end of the queue; non-exclusive waiters to the front.

### Common Patterns

The typical sleep pattern is:

```c
DEFINE_WAIT(wait);
add_wait_queue(&wq_head, &wait);
while (!condition) {
    set_current_state(TASK_INTERRUPTIBLE);
    schedule();
    if (signal_pending(current))
        break;  /* handle signal */
}
set_current_state(TASK_RUNNING);
remove_wait_queue(&wq_head, &wait);
```

Helper macros like `wait_event()` and `wait_event_interruptible()` encapsulate this pattern.

## PID Management

### pidmap_array

The kernel allocates PIDs from a bitmap called `pidmap_array`. Each bit represents one PID value, allowing O(1) allocation and freeing. The default maximum PID is 32768 (fitting in a single page), but it can be increased to over 4 million via `/proc/sys/kernel/pid_max`. PIDs are allocated sequentially with wraparound, minimizing short-term PID reuse.

### PID Hash Tables

For fast PID-to-`task_struct` lookup, the kernel maintains four hash tables (one for each PID type: `PIDTYPE_PID`, `PIDTYPE_TGID`, `PIDTYPE_PGID`, `PIDTYPE_SID`). These use chaining for collision resolution. The hash tables enable efficient operations like "send a signal to all processes in process group X" without scanning the entire task list.

## User-Space Process Programming

From the application developer's perspective (detailed in [src-advanced-linux-programming](../sources/src-advanced-linux-programming.md)):

### The exec Family

After `fork()`, the child typically calls one of the `exec` functions to replace its process image with a new program:

| Function | Args style | PATH search? | Environment |
|----------|-----------|--------------|-------------|
| `execl` | Variadic list | No | Inherited |
| `execlp` | Variadic list | Yes | Inherited |
| `execle` | Variadic list | No | Explicit `envp` |
| `execv` | Array | No | Inherited |
| `execvp` | Array | Yes | Inherited |
| `execve` | Array | No | Explicit `envp` (the underlying syscall) |

All `exec` functions replace the current process image entirely — the calling code does not continue after a successful `exec`. On failure, `exec` returns -1 and the calling code continues.

### Zombie Prevention Strategies

1. **`waitpid()` in parent** — actively reap children when convenient. Use `WNOHANG` for non-blocking check.
2. **`SIGCHLD` handler** — install a handler that calls `waitpid(-1, NULL, WNOHANG)` in a loop.
3. **`SA_NOCLDWAIT`** — set this flag on the `SIGCHLD` action to tell the kernel not to create zombies.
4. **Double fork** — fork, let the first child fork again and exit immediately. The grandchild is orphaned to init, which reaps it automatically. The parent only needs to wait for the short-lived intermediate child.

### Process Termination

- **`exit(status)`** — runs `atexit()` handlers, flushes stdio buffers, then calls `_exit()`.
- **`_exit(status)`** — immediate termination without cleanup. Used in child processes after `fork()` when `exec()` fails, to avoid double-flushing stdio buffers.
- **`abort()`** — sends `SIGABRT` to self, producing a core dump if not caught.

## See also

- [process-scheduler](process-scheduler.md)
- [pthreads](pthreads.md)
- [signals](signals.md)
- [system-calls](system-calls.md)
- [memory-management](memory-management.md)
- [interrupt-handling](interrupt-handling.md)
- [proc-filesystem](proc-filesystem.md)
