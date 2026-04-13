---
type: entity
created: 2026-04-08
updated: 2026-04-09
sources: [understanding-the-linux-kernel, advanced-linux-programming]
tags: [system-calls, x86, user-kernel-interface, strace]
---

# System Calls

System calls are the fundamental interface between user-space applications and the Linux kernel. They provide controlled entry points into the kernel for operations that require privileged access — file I/O, process creation, memory allocation, network communication, and more. Linux 2.6.11 on x86 defines approximately 290 system calls.

## POSIX API vs. System Calls

It is important to distinguish between a **POSIX API function** and a **system call**:

- A **POSIX API** (e.g., `malloc()`, `printf()`, `fopen()`) is a C library function that may or may not invoke a system call. `malloc()` manages a user-space heap and only calls `brk()` or `mmap()` when it needs more memory from the kernel. `printf()` buffers output and only calls `write()` when the buffer is flushed.
- A **system call** (e.g., `read()`, `write()`, `fork()`, `execve()`) is a specific request to the kernel, identified by a unique number, with a defined calling convention.

The C library (glibc) provides wrapper functions for each system call that handle the low-level mechanics of entering the kernel. User programs typically call these wrappers rather than invoking system calls directly.

## x86 Entry Mechanism

### The Classic Path: int $0x80

The traditional x86 mechanism for entering kernel mode:

1. User-space code loads the **system call number** into the `eax` register.
2. Up to six parameters are loaded into registers: `ebx`, `ecx`, `edx`, `esi`, `edi`, `ebp` (in that order).
3. The `int $0x80` instruction triggers software interrupt 128.
4. The CPU automatically:
   - Switches to the kernel stack (loads `ss` and `esp` from the TSS).
   - Pushes the user-space `ss`, `esp`, `eflags`, `cs`, and `eip` onto the kernel stack.
   - Loads `cs:eip` from IDT entry 128, which points to the `system_call()` handler.

### The Fast Path: sysenter / sysexit

Pentium II and later processors provide the `sysenter` / `sysexit` instruction pair, which are significantly faster than `int $0x80` because they avoid the overhead of the interrupt descriptor table lookup, privilege-level checks, and stack switch logic:

- `sysenter` transitions to kernel mode (ring 0) by loading predefined values from MSRs (Model-Specific Registers) into `cs`, `eip`, `ss`, and `esp`. The kernel sets these MSRs during boot to point to the `sysenter_entry` handler.
- `sysexit` performs the reverse transition back to user mode (ring 3).

The `sysenter` path is approximately 45% faster than `int $0x80` on some processors.

### The vsyscall Page

The kernel maps a special read-only page into every process's address space at a fixed address (the **vsyscall page**, also called the **vDSO** — virtual Dynamic Shared Object). This page contains:

1. The correct system call entry code for the current processor. On CPUs supporting `sysenter`, the page contains `sysenter`-based entry; on older CPUs, it uses `int $0x80`. User-space code calls a function on this page without needing to know which mechanism to use.
2. **Fast system calls** that can be executed entirely in user space. For example, `gettimeofday()` can read a kernel-maintained clock value directly from the vsyscall page without transitioning to kernel mode, avoiding the cost of a full system call.

## System Call Dispatch

### system_call() Entry Point

The `system_call()` assembly handler (in `arch/i386/kernel/entry.S`) is the common entry point for both `int $0x80` and `sysenter` paths (they converge after initial register setup). Its flow:

1. **Save registers**: Pushes `eax` (syscall number) and the other general-purpose registers onto the kernel stack, building a `pt_regs` structure.
2. **Validate syscall number**: Checks that `eax` is less than `NR_syscalls` (the total number of implemented system calls). If invalid, returns `-ENOSYS`.
3. **Dispatch**: Calls `sys_call_table[eax]` — an indirect function call through the system call table.
4. **Store return value**: Places the return value from the system call function into `eax` on the saved register frame (so it is returned to user space).
5. **Check work flags** before returning to user space (see "Exit Path" below).

### sys_call_table

The `sys_call_table` is a statically-defined array of function pointers, indexed by system call number. For example:

| Index (eax) | Function | Purpose |
|-------------|----------|---------|
| 0 | `sys_restart_syscall` | Restart a system call after signal |
| 1 | `sys_exit` | Terminate the calling process |
| 2 | `sys_fork` | Create a child process |
| 3 | `sys_read` | Read from a file descriptor |
| 4 | `sys_write` | Write to a file descriptor |
| 5 | `sys_open` | Open a file |
| ... | ... | ... |
| 120 | `sys_clone` | Create a child process with flags |

Each `sys_*` function is implemented in C in the appropriate kernel subsystem.

## Parameter Passing and Verification

### Register Convention

System call parameters are passed in registers (up to 6):

| Parameter | Register |
|-----------|----------|
| 1st | `ebx` |
| 2nd | `ecx` |
| 3rd | `edx` |
| 4th | `esi` |
| 5th | `edi` |
| 6th | `ebp` |

The return value is placed in `eax`. A negative return value in the range `-1` to `-4095` indicates an error; the C library wrapper negates it and stores it in `errno`.

System calls that need more than 6 parameters (very rare) pass a pointer to a structure in user memory.

### Address Verification

When a system call receives a pointer to user-space memory, the kernel **must** verify that:

1. The address is in the user portion of the address space (below `PAGE_OFFSET`, i.e., below 3 GB on a default x86 configuration).
2. The memory is actually accessible (mapped and with correct permissions).

#### access_ok()

The `access_ok(type, addr, size)` macro performs the first check — it verifies that the address range falls within user space. It does not check page-table permissions (that would be too expensive). It takes:

- `type`: `VERIFY_READ` or `VERIFY_WRITE`.
- `addr`: Starting address.
- `size`: Number of bytes.

This is a quick sanity check against a user trying to trick the kernel into reading/writing kernel memory.

#### Data Transfer Functions

For actually copying data between user and kernel space:

| Function | Direction | Notes |
|----------|-----------|-------|
| `get_user(x, ptr)` | User -> Kernel | Reads a simple value (1, 2, or 4 bytes) from user space into kernel variable `x`. |
| `put_user(x, ptr)` | Kernel -> User | Writes a simple value from kernel to user space. |
| `copy_from_user(to, from, n)` | User -> Kernel | Copies `n` bytes from user address `from` to kernel buffer `to`. |
| `copy_to_user(to, from, n)` | Kernel -> User | Copies `n` bytes from kernel buffer `from` to user address `to`. |

These functions use the **fixup mechanism**: the actual memory access is performed, and if it triggers a page fault at an address that is not valid for the process, the page fault handler looks up the faulting instruction in a special exception table (`__ex_table`). If found, it redirects execution to a fixup routine that returns an error (`-EFAULT`) instead of crashing the kernel.

## Entry and Exit Path Details

### Full Entry Path

When a system call is invoked:

1. User-space: `glibc` wrapper loads registers, executes `sysenter` or `int $0x80`.
2. CPU: switches to kernel stack, saves user-space registers.
3. `system_call()` / `sysenter_entry()`: saves remaining registers (building `pt_regs`).
4. Dispatches to `sys_call_table[eax]`.
5. The `sys_*` function executes, possibly sleeping, allocating memory, accessing hardware, etc.

### Exit Path

After the system call returns, before transitioning back to user mode, the kernel checks several flags in `thread_info->flags`:

| Flag | Check | Action |
|------|-------|--------|
| `TIF_NEED_RESCHED` | Is rescheduling needed? | Call `schedule()` to give the CPU to a higher-priority process. After `schedule()` returns, re-check all flags. |
| `TIF_SIGPENDING` | Are there pending [signals](signals.md)? | Call `do_signal()` to deliver them. Signal delivery may involve setting up a user-space signal handler frame. |
| `TIF_SYSCALL_TRACE` | Is `ptrace` tracing syscalls? | Notify the tracer (used by `strace`). |

This exit-path work is critical: it is the point where signals are delivered and where the [process-scheduler](process-scheduler.md) can preempt the current process.

## System Call Restart After Signals

When a process is sleeping inside a system call (e.g., waiting for data on a socket) and a signal arrives, the system call may be interrupted. The kernel must decide whether to:

1. **Return an error** (`-EINTR`) to user space.
2. **Automatically restart** the system call after signal handling.

This is controlled by the **error code** the interrupted system call uses and the **SA_RESTART** flag on the signal handler:

| Error Code | Meaning | SA_RESTART set? | Result |
|------------|---------|-----------------|--------|
| `-ERESTARTSYS` | Restartable; respect SA_RESTART | Yes | Kernel automatically restarts the syscall. |
| `-ERESTARTSYS` | Restartable; respect SA_RESTART | No | Returns `-EINTR` to user space. |
| `-ERESTARTNOINTR` | Always restart | (ignored) | Kernel always restarts the syscall (used for `nanosleep`, etc.). |
| `-ERESTARTNOHAND` | Restart only if no handler | Handler exists | Returns `-EINTR`. |
| `-ERESTARTNOHAND` | Restart only if no handler | Default/ignore action | Kernel restarts the syscall. |

To restart, the kernel rewinds the user-space instruction pointer (`eip`) to point back to the `sysenter` / `int $0x80` instruction and restores `eax` to the original system call number, so the system call is re-executed transparently when the process returns to user mode after handling the signal.

## ptrace and System Call Tracing

The `ptrace()` system call allows one process (the tracer) to observe and control the execution of another (the tracee). When system call tracing is enabled (`PTRACE_SYSCALL`):

1. At system call entry, if `TIF_SYSCALL_TRACE` is set, the kernel stops the tracee and notifies the tracer. The tracer can inspect and modify the system call number and arguments.
2. The system call executes.
3. At system call exit, the kernel again stops the tracee, allowing the tracer to inspect and modify the return value.

This mechanism is the foundation of tools like `strace`, which displays every system call a process makes, and `ltrace`, which traces library calls.

The tracee is placed in `TASK_TRACED` state while stopped for the tracer, which is distinct from `TASK_STOPPED` (used for job control). See [process-management](process-management.md) for details on process states.

## Adding a New System Call

To add a new system call to the kernel:

1. Add an entry to `sys_call_table` in `arch/i386/kernel/entry.S`.
2. Define the new `sys_*` function, following kernel conventions (arguments from registers, return `-errno` on failure).
3. Increment `NR_syscalls`.
4. Add a `#define __NR_newsyscall` to `include/asm-i386/unistd.h`.
5. Export a user-space wrapper (typically via glibc).

However, the kernel community generally prefers extending existing multiplexer system calls (e.g., `ioctl()`, `fcntl()`, `prctl()`) or using new file-based interfaces (e.g., in `/proc` or `/sys`) over adding new system call numbers.

## Commonly Used System Calls (User-Space Reference)

Beyond the kernel-level mechanics above, the following system calls are frequently used by application developers (detailed in [src-advanced-linux-programming](../sources/src-advanced-linux-programming.md)):

| System call | Purpose |
|-------------|---------|
| `access(path, mode)` | Check file accessibility (`R_OK`, `W_OK`, `X_OK`, `F_OK`) without opening. Subject to TOCTOU races — prefer attempting the operation directly |
| `fcntl(fd, cmd, ...)` | Manipulate file descriptors: duplicate (`F_DUPFD`), get/set flags (`O_NONBLOCK`, `FD_CLOEXEC`), advisory record locking (`F_SETLK`, `F_SETLKW`) |
| `fsync(fd)` / `fdatasync(fd)` | Flush file to disk. `fsync` flushes data + metadata; `fdatasync` flushes data only (faster) |
| `getrlimit` / `setrlimit` | Query/modify resource limits: max open files, CPU time, file size, stack size, max processes |
| `getrusage(who, &usage)` | Resource usage statistics: user/system CPU time, page faults, context switches, max RSS |
| `mmap(addr, len, prot, flags, fd, offset)` | Map files or anonymous memory. `MAP_SHARED` for IPC, `MAP_PRIVATE` for COW copy, `MAP_ANONYMOUS` for heap-like allocation |
| `mlock(addr, len)` | Lock pages in RAM (prevent swapping). Requires `CAP_IPC_LOCK`. Used for sensitive data (cryptographic keys) or real-time applications |
| `sendfile(out_fd, in_fd, &offset, count)` | Zero-copy file-to-socket transfer. The kernel moves data directly between file descriptors without user-space buffering |
| `setitimer(which, &new, &old)` | Interval timers: `ITIMER_REAL` (wall clock, delivers `SIGALRM`), `ITIMER_VIRTUAL` (user CPU, `SIGVTALRM`), `ITIMER_PROF` (user+sys CPU, `SIGPROF`) |
| `sysinfo(&info)` | System-wide stats: uptime, load averages, RAM total/free/shared/buffered, swap total/free, process count |

### strace

`strace` is the primary diagnostic tool for understanding system call behavior. It intercepts every system call made by a process (via `ptrace`) and prints the call name, arguments, and return value:

```
$ strace -e trace=open,read,write ./myprogram
open("/etc/passwd", O_RDONLY)           = 3
read(3, "root:x:0:0:root:/root:/bin/bash\n"..., 4096) = 1456
write(1, "Hello\n", 6)                 = 6
```

Key strace options: `-p pid` (attach to running process), `-f` (follow forks), `-e trace=category` (filter by syscall category: `file`, `process`, `network`, `signal`, `ipc`), `-c` (summary statistics), `-t` (timestamps).

## See also

- [process-management](process-management.md)
- [process-scheduler](process-scheduler.md)
- [signals](signals.md)
- [interrupt-handling](interrupt-handling.md)
- [memory-management](memory-management.md)
- [proc-filesystem](proc-filesystem.md)
- [concept-linux-security](../concepts/concept-linux-security.md)
