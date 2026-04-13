---
type: entity
created: 2026-04-08
updated: 2026-04-09
sources: [understanding-the-linux-kernel, advanced-linux-programming]
tags: [signals, ipc, process-management]
---

# Signals

Signals are the oldest inter-process communication mechanism in Unix. They provide asynchronous notifications to processes about events such as hardware exceptions, terminal input, timers, child process termination, and explicit requests from other processes. Linux supports both the classic POSIX signals and the POSIX.1b real-time signal extensions.

## Signal Types

### Regular Signals (1--31)

The first 31 signals are the traditional POSIX signals. Key characteristics:

- Represented as a **bitmask** in a `sigset_t` (a single `unsigned long` on 32-bit architectures, or an array of `unsigned long` on 64-bit).
- **Not queued**: if the same signal is sent multiple times before the process handles it, only one instance is recorded. The bit is set on the first delivery and remains set; subsequent sends are silently dropped.
- Each signal has a fixed meaning defined by POSIX.

Common signals (selected):

| Signal | # | Default | Cause |
|--------|---|---------|-------|
| `SIGINT` | 2 | Terminate | Ctrl+C |
| `SIGQUIT` | 3 | Core dump | Ctrl+\ |
| `SIGILL` | 4 | Core dump | Illegal instruction |
| `SIGABRT` | 6 | Core dump | `abort()` |
| `SIGKILL` | 9 | Terminate | Unconditional kill (uncatchable) |
| `SIGSEGV` | 11 | Core dump | Invalid memory reference |
| `SIGPIPE` | 13 | Terminate | Broken pipe |
| `SIGALRM` | 14 | Terminate | `alarm()` timer |
| `SIGTERM` | 15 | Terminate | Polite termination |
| `SIGCHLD` | 17 | Ignore | Child stopped/terminated |
| `SIGSTOP` | 19 | Stop | Unconditional stop (uncatchable) |
| `SIGTSTP` | 20 | Stop | Ctrl+Z |

### Real-Time Signals (32--64)

POSIX.1b real-time signals extend the signal mechanism:

- **Queued**: multiple instances of the same real-time signal are kept in a queue and delivered individually. Each delivery includes associated data.
- **Ordered**: real-time signals are delivered in signal-number order (lower number = higher priority).
- **No predefined meaning**: their semantics are application-defined.
- Sent via `sigqueue()` rather than `kill()`, allowing a `sigval` (integer or pointer) to be passed along with the signal.

Internally, queued signals are stored as `struct sigqueue` entries in a linked list, rather than as bits in a bitmask.

## Default Signal Actions

Every signal has a default action: **Terminate** (SIGHUP, SIGINT, SIGPIPE, SIGTERM, etc.), **Core dump** (SIGQUIT, SIGILL, SIGABRT, SIGFPE, SIGSEGV, SIGBUS), **Stop** (SIGSTOP, SIGTSTP, SIGTTIN, SIGTTOU), **Ignore** (SIGCHLD, SIGURG, SIGWINCH), or **Continue** (SIGCONT).

`SIGKILL` and `SIGSTOP` **cannot be caught, blocked, or ignored**. The kernel enforces this unconditionally — `sigaction()` returns `-EINVAL` for handler installation, and `sigprocmask()` silently removes them from blocked sets.

## Key Data Structures

### signal_struct

Shared by all threads in a thread group (pointed to by `task_struct->signal`). Contains:

- `shared_pending` — a `struct sigpending` holding signals sent to the thread group as a whole (e.g., via `kill(pid, sig)` where `pid` is the TGID).
- `group_exit_code` — exit status for the thread group.
- Resource limits, timers, and accounting information shared across threads.

### sighand_struct

Also shared by all threads in a thread group (pointed to by `task_struct->sighand`). Contains:

- `action[_NSIG]` — an array of `struct k_sigaction`, one per signal number. Each entry specifies the handler function, flags (`SA_RESTART`, `SA_SIGINFO`, `SA_NODEFER`, etc.), and the signal mask to apply during handler execution.
- Reference count and a spinlock.

### sigpending

Holds the set of pending signals for a task or thread group:

- `signal` — a `sigset_t` bitmask indicating which signals are pending.
- `list` — a linked list of `struct sigqueue` entries for queued signals (real-time signals and signals sent via `sigqueue()`).

Each thread has its own `sigpending` (`task_struct->pending`) for signals sent directly to that thread, plus access to the thread group's `signal->shared_pending`.

### siginfo_t

Carries metadata about a signal delivery:

- `si_signo` — signal number.
- `si_errno` — associated error code (if any).
- `si_code` — origin of the signal (`SI_USER`, `SI_KERNEL`, `SI_QUEUE`, `SI_TIMER`, etc.).
- A union of additional data depending on the signal type: PID and UID of the sender, child exit status, faulting address (for `SIGSEGV`/`SIGBUS`), timer ID, etc.

### k_sigaction

The kernel's internal representation of a signal disposition:

- `sa_handler` / `sa_sigaction` — pointer to the handler function (or `SIG_DFL` / `SIG_IGN`).
- `sa_flags` — behavioral flags:
  - `SA_RESTART` — automatically restart interrupted system calls.
  - `SA_SIGINFO` — use the three-argument `sa_sigaction` handler that receives `siginfo_t`.
  - `SA_NODEFER` — do not block the signal while its handler is running.
  - `SA_RESETHAND` — reset the handler to `SIG_DFL` after first delivery (one-shot).
  - `SA_NOCLDSTOP` — do not send `SIGCHLD` when a child stops (only on exit).
  - `SA_NOCLDWAIT` — do not create zombie children.
- `sa_mask` — additional signals to block while the handler executes.

## Signal Generation

Signal generation is the act of making a signal pending for a process or thread group.

### Key Generation Functions

| Function | Scope | Description |
|----------|-------|-------------|
| `send_sig_info(sig, info, task)` | Single thread | Sends signal `sig` to a specific task. Core low-level function. |
| `force_sig(sig, task)` | Single thread | Forces a signal that cannot be ignored or blocked. Used for hardware exceptions (e.g., kernel sends `SIGSEGV` when a process accesses unmapped memory). If the signal is blocked or ignored, the kernel resets the handler to `SIG_DFL` and unblocks it. |
| `group_send_sig_info(sig, info, task)` | Thread group | Sends a signal to an entire thread group. This is what `kill()` uses — the signal is placed in `shared_pending`. |
| `send_sig_queue(sig, queue_entry, task)` | Single thread | Sends a queued (real-time) signal with associated data. |
| `specific_send_sig_info(sig, info, task)` | Single thread | Optimized version of `send_sig_info()` used when the caller already holds the `sighand` lock. |

### Generation Flow

When `send_sig_info()` is called:

1. Checks if the signal is being ignored (handler is `SIG_IGN` or handler is `SIG_DFL` and default action is ignore). If so, returns immediately.
2. Checks if the signal is blocked and not `SIGKILL`/`SIGSTOP`.
3. For regular signals: sets the bit in `task->pending.signal`. If the bit was already set, the signal is collapsed (no queuing).
4. For real-time signals or signals sent via `sigqueue()`: allocates a `struct sigqueue`, fills in the `siginfo_t`, and adds it to the `task->pending.list`.
5. Calls `signal_wake_up()`, which sets `TIF_SIGPENDING` in the target task's `thread_info` and, if the task is sleeping in `TASK_INTERRUPTIBLE` state, wakes it up.

## Signal Delivery

Signal delivery is the act of causing the process to respond to a pending signal. It occurs on the return path from kernel mode to user mode — after a system call, interrupt, or exception.

### do_signal()

Called from the system call exit path when `TIF_SIGPENDING` is set. It loops, calling `dequeue_signal()` to fetch the next pending signal:

1. `dequeue_signal()` selects the lowest-numbered pending signal (checking per-thread `pending` before thread-group `shared_pending`). It clears the corresponding bit in the pending bitmask and removes any associated `sigqueue` entry.
2. If the signal disposition is `SIG_IGN`, it is discarded and the loop continues.
3. If the disposition is `SIG_DFL`, the default action is performed:
   - **Terminate**: calls `do_group_exit()`.
   - **Core dump**: generates a core dump file, then terminates.
   - **Stop**: sets the task state to `TASK_STOPPED` and calls `schedule()`.
   - **Ignore**: discards the signal.
   - **Continue**: wakes stopped tasks (handled earlier during generation).
4. If a user handler is installed, calls `handle_signal()`.

### handle_signal() and Stack Frame Setup

When delivering a signal to a user-space handler, the kernel must arrange for the handler to execute in user mode and then cleanly return to whatever the process was doing before. This is accomplished by manipulating the user-space stack:

#### setup_frame() / setup_rt_frame()

These functions build a **signal frame** on the user-space stack:

```
User stack before signal:
+------------------+
|  (normal stack)  |
+------------------+  <-- user esp

User stack after setup_frame():
+------------------+
|  (normal stack)  |
+------------------+
|  sigreturn code  |  <-- small trampoline: mov __NR_sigreturn, %eax; int $0x80
+------------------+
|  saved sigset    |  <-- blocked signal mask to restore
+------------------+
|  saved pt_regs   |  <-- full register state at time of signal
+------------------+
|  signal number   |  <-- argument to handler
+------------------+  <-- new user esp, eip set to handler address
```

For `SA_SIGINFO` handlers, `setup_rt_frame()` is used instead, which additionally pushes a `siginfo_t` and a `ucontext_t` (containing the full machine context), and the trampoline calls `rt_sigreturn` instead of `sigreturn`.

The kernel then modifies the saved registers (`pt_regs`) so that when the process returns to user mode:
- `eip` points to the signal handler function.
- `esp` points to the constructed stack frame.
- The handler's arguments are in the correct positions on the stack.

### sigreturn() / rt_sigreturn()

When the signal handler function returns (via `ret`), it executes the trampoline code on the stack, which invokes the `sigreturn()` (or `rt_sigreturn()`) system call. This system call:

1. Reads the saved register state (`pt_regs`) from the signal frame on the user stack.
2. Restores the process's register state to what it was before the signal was delivered.
3. Restores the blocked signal mask.
4. Returns to the original interrupted code path as if the signal never happened.

## Blocked Signals

Each thread has a **blocked signal mask** (`task_struct->blocked`), a `sigset_t` bitmask indicating which signals are currently blocked. Blocked signals:

- Are not delivered (handlers are not invoked).
- Remain pending until unblocked.
- For regular signals, if the same signal is sent multiple times while blocked, only one instance is recorded (bitmask semantics).
- For real-time signals, each instance is queued.

The `sigprocmask()` system call manipulates the blocked mask (`SIG_BLOCK`, `SIG_UNBLOCK`, `SIG_SETMASK`).

### recalc_sigpending()

After any change to the blocked mask or the pending signal set, the kernel calls `recalc_sigpending()`. This function checks whether there are any pending signals that are not blocked, and sets or clears the `TIF_SIGPENDING` flag in `thread_info` accordingly. This flag is what the system call exit path checks to decide whether to call `do_signal()`.

## SA_RESTART and System Call Interaction

When a signal interrupts a process that is blocked inside a [system call](system-calls.md), the interaction between signal handling and system call restart is controlled by:

1. The **error code** used by the system call when interrupted (see [system-calls](system-calls.md) for the full table):
   - `-ERESTARTSYS` — restart if `SA_RESTART` is set on the signal's handler.
   - `-ERESTARTNOINTR` — always restart regardless of handler flags.
   - `-ERESTARTNOHAND` — restart only if the signal's action is default or ignore (not a user handler).
2. The `SA_RESTART` flag on the signal handler's `k_sigaction`.

If a restart is appropriate, the kernel rewinds `eip` to re-execute the system call entry instruction and restores `eax` to the original system call number. The restart is transparent to user space.

If the system call is not restarted, user space receives `-EINTR` in `errno`.

## Thread Group Signal Handling

In a multi-threaded process (thread group), signal handling follows these rules:

### Per-Thread vs. Group Signals

- **Thread-directed signals** (e.g., sent via `tkill()` or `tgkill()`, or generated by hardware exceptions like `SIGSEGV`): placed in the target thread's `task->pending` and delivered to that specific thread.
- **Process-directed signals** (e.g., sent via `kill()` to the TGID): placed in `task->signal->shared_pending` and delivered to **any one thread** in the group that has the signal unblocked.

### Thread Selection for Group Signals

When a group-directed signal arrives, the kernel selects a target thread:

1. Prefer the thread that is currently running (or was last running) if it has the signal unblocked.
2. Otherwise, iterate through threads in the group to find one with the signal unblocked.
3. If all threads have the signal blocked, it remains in `shared_pending` until some thread unblocks it.

This is why POSIX requires that `pthread_sigmask()` be used carefully: a signal sent to the process is delivered to an arbitrary thread that has not blocked it. Common practice is to block signals in all threads except one dedicated signal-handling thread.

### Fatal Signals and Thread Groups

When a fatal signal (e.g., `SIGKILL`, unhandled `SIGSEGV`) targets a thread group:

1. All threads in the group are terminated.
2. `do_group_exit()` sets a flag and sends `SIGKILL` to every thread in the group.
3. Each thread terminates during its next return to user mode.

## Job Control Signals

Job control allows a shell to manage groups of processes (jobs) running in the foreground or background. The relevant signals:

| Signal | Purpose |
|--------|---------|
| `SIGSTOP` | Stop a process unconditionally. Cannot be caught. Used by `kill -STOP`. |
| `SIGTSTP` | "Terminal stop" — sent by Ctrl+Z. Can be caught (some programs clean up before stopping). |
| `SIGTTIN` | Sent when a background process attempts to read from its controlling terminal. Stops the process. |
| `SIGTTOU` | Sent when a background process attempts to write to its controlling terminal (if `TOSTOP` is set). Stops the process. |
| `SIGCONT` | Resume a stopped process. Has special handling: when generated, it automatically discards any pending stop signals (`SIGSTOP`, `SIGTSTP`, `SIGTTIN`, `SIGTTOU`). Conversely, generating a stop signal discards any pending `SIGCONT`. |

When a process stops (enters `TASK_STOPPED`), the kernel sends `SIGCHLD` to its parent (unless `SA_NOCLDSTOP` is set), informing the parent of the status change. The parent can then use `waitpid()` with `WUNTRACED` to detect the stop.

## Uncatchable Signals

`SIGKILL` (signal 9) and `SIGSTOP` (signal 19) receive special treatment throughout the kernel:

- `sigaction()` / `rt_sigaction()`: returns `-EINVAL` if the process tries to install a handler for either signal.
- `sigprocmask()`: silently removes `SIGKILL` and `SIGSTOP` from any mask the process attempts to set as its blocked mask.
- `force_sig()` for `SIGKILL`: if a process somehow has `SIGKILL` blocked (kernel bug or special case), `force_sig()` forcibly unblocks it and resets the action to `SIG_DFL`.

This guarantees that the system administrator can always kill or stop any process, regardless of what the process's signal handlers are doing.

## User-Space Signal API

From the application developer's perspective (covered in detail in [src-advanced-linux-programming](../sources/src-advanced-linux-programming.md)):

### signal() vs. sigaction()

- **`signal(sig, handler)`** — the classic, simplified API. Its behavior varies across Unix implementations (on some systems the handler is reset to `SIG_DFL` after each delivery). Non-portable for anything beyond `SIG_IGN` and `SIG_DFL`.
- **`sigaction(sig, &act, &oldact)`** — the reliable, portable API. Provides full control over handler flags, signal masking during handler execution, and `siginfo_t` delivery. Always prefer `sigaction()` in new code.

### Common Signal Patterns

- **Child reaping**: install a `SIGCHLD` handler that calls `waitpid(-1, &status, WNOHANG)` in a loop to reap all terminated children. Alternatively, set `SA_NOCLDWAIT` to prevent zombies without a handler.
- **Graceful shutdown**: catch `SIGTERM` and `SIGINT` to set a flag, then check the flag in the main loop to clean up and exit. Never call non-async-signal-safe functions (like `printf`, `malloc`) from a signal handler.
- **Double-fork trick**: fork twice, with the intermediate process exiting immediately, to orphan the grandchild to init — avoids zombie accumulation without signal handling.

### Async-Signal Safety

Signal handlers execute asynchronously, interrupting the program at arbitrary points. Only a limited set of functions are safe to call from handlers (the POSIX async-signal-safe functions): `write()`, `_exit()`, `signal()`, and a few dozen others. Notably, `printf()`, `malloc()`, `free()`, and most C library functions are **not** safe. The recommended pattern is to set a `volatile sig_atomic_t` flag in the handler and check it in the main program flow.

## See also

- [process-management](process-management.md)
- [pthreads](pthreads.md)
- [system-calls](system-calls.md)
- [process-scheduler](process-scheduler.md)
- [interrupt-handling](interrupt-handling.md)
