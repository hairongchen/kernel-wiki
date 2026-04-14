---
type: source
title: "Advanced Linux Programming"
created: 2026-04-09
updated: 2026-04-09
sources: [advanced-linux-programming]
tags: [reference-book, linux-programming, userspace, gnu-linux, pthreads, ipc, security]
---

# Advanced Linux Programming

**Authors:** Mark Mitchell, Jeffrey Oldham, Alex Samuel
**Publisher:** New Riders Publishing, June 2001
**Covers:** GNU/Linux application development from the programmer's perspective
**ISBN:** 0-7357-1043-0

## Overview

A practical guide to writing robust, high-performance applications on GNU/Linux. Unlike the kernel-internals books in this wiki, ALP approaches Linux from the **application developer's perspective** — it covers how to use the system call interfaces, threading libraries, IPC mechanisms, and security features that the kernel provides. This makes it a valuable complement to kernel-level sources: where "Understanding the Linux Kernel" explains *how* `fork()` or `pipe()` works inside the kernel, ALP explains *how to use them correctly* from user space, including common pitfalls, patterns, and best practices.

The book targets C/C++ developers and assumes familiarity with one of these languages. It covers GNU/Linux-specific extensions beyond POSIX where relevant.

## Chapter Summaries

### Ch 1: Getting Started

Introduces the GNU/Linux development toolchain:

- **GCC** — compilation pipeline (preprocessing → compilation → assembly → linking), optimization levels (`-O1`/`-O2`), warning flags (`-Wall`, `-Werror`, `-pedantic`), static vs. shared libraries (`ar` for `.a`, `-shared -fPIC` for `.so`).
- **GNU Make** — dependency-driven build automation. Pattern rules, automatic variables (`$@`, `$<`, `$^`), phony targets. Standard Makefile conventions.
- **GDB** — debugging: breakpoints, watchpoints, stack inspection, core dump analysis. Running processes under GDB, attaching to running processes.
- **Emacs** — briefly introduced as an IDE-like development environment with GDB integration.

### Ch 2: Writing Good GNU/Linux Software

Essential patterns for well-behaved Linux programs:

- **Program interaction** — command-line argument parsing with `getopt_long()`, environment variable access via `environ` and `getenv()`/`setenv()`, temporary file creation (`mkstemp()`).
- **Standard I/O** — the `stdin`/`stdout`/`stderr` trio, format codes, error output conventions.
- **Error handling** — `errno` and `strerror()`, systematic error reporting, assert for invariants vs. runtime error handling for recoverable conditions.
- **Resource management** — cleanup patterns, `atexit()` handlers.

### Ch 3: Processes

Comprehensive coverage of the Unix process model from the programmer's perspective:

- **Process creation with `fork()`** — returns 0 to child, child PID to parent. The child is an (almost) exact copy of the parent. Typical fork-then-exec pattern.
- **The `exec` family** — `execl`, `execlp`, `execle`, `execv`, `execvp`, `execve`: variations for how arguments and environment are passed. The exec'd program replaces the calling process image entirely.
- **Process termination** — `exit()` vs. `_exit()`, `abort()`, `SIGTERM`/`SIGKILL` handling.
- **Waiting for children** — `wait()`, `waitpid()`, `WIFEXITED`/`WIFSIGNALED`/`WIFSTOPPED` macros, `WNOHANG` for non-blocking wait.
- **Zombie processes** — arise when a child exits before the parent waits. Prevention strategies: double-fork trick, `SIGCHLD` handler with `wait()`, `SA_NOCLDWAIT`.
- **Signals** (overview) — signal as asynchronous notification, `signal()` vs. `sigaction()` (prefer `sigaction()`), common signals table, `SIGCHLD` for child reaping.

See [process-management](../entities/process-management.md), [signals](../entities/signals.md).

### Ch 4: Threads

In-depth coverage of POSIX threads (`pthreads`):

- **Thread creation** — `pthread_create()` with function pointer and `void*` argument. Return value retrieval via `pthread_join()`. Detached threads via `pthread_detach()` or `PTHREAD_CREATE_DETACHED` attribute.
- **Thread cancellation** — `pthread_cancel()`, cancellation types (deferred vs. asynchronous), cancellation points, cleanup handlers with `pthread_cleanup_push()`/`pthread_cleanup_pop()`.
- **Thread-specific data** — `pthread_key_create()`, `pthread_setspecific()`/`pthread_getspecific()` for per-thread globals. Destructor functions for automatic cleanup.
- **Synchronization primitives**:
  - **Mutexes** — `pthread_mutex_lock()`/`unlock()`, `PTHREAD_MUTEX_INITIALIZER`, deadlock avoidance (lock ordering, trylock).
  - **Semaphores** — POSIX unnamed semaphores: `sem_init()`, `sem_wait()`, `sem_post()`, `sem_destroy()`.
  - **Condition variables** — `pthread_cond_wait()`, `pthread_cond_signal()`, `pthread_cond_broadcast()`. Always pair with a mutex and a predicate loop ("spurious wakeup" protection).
- **Thread safety** — reentrant functions, the `_r` suffix convention (e.g., `strtok_r()`), thread-safe library design.
- **Thread vs. process trade-offs** — threads share address space (faster communication, less isolation), processes have separate address spaces (more robust, `fork()`+`exec()` overhead).

See [pthreads](../entities/pthreads.md).

### Ch 5: Interprocess Communication

All major IPC mechanisms available to Linux applications:

- **Shared memory** — `shmget()`, `shmat()`, `shmdt()`, `shmctl()`. Fastest IPC (no data copying after setup). Requires explicit synchronization (semaphores or mutexes in shared region). `IPC_PRIVATE` for related processes, key-based for unrelated.
- **Memory-mapped files** — `mmap()` with `MAP_SHARED` for IPC via a backing file. `msync()` for durability. Provides both IPC and file I/O in one mechanism.
- **Pipes** — `pipe()` creating `fd[0]` (read) and `fd[1]` (write). Unidirectional, between related processes. Used with `fork()` for parent-child communication. `dup2()` for redirecting `stdin`/`stdout` through pipes (shell pipeline pattern).
- **FIFOs (named pipes)** — `mkfifo()`. Like pipes but with a filesystem name, allowing communication between unrelated processes.
- **UNIX domain sockets** — `socket(PF_UNIX, ...)` with `AF_UNIX` address family. Bidirectional, connection-oriented or datagram. `socketpair()` for related processes. Used extensively for local client-server communication (e.g., X11, D-Bus, syslog).

See [ipc](../entities/ipc.md).

### Ch 6: Devices

How Linux represents hardware as files:

- **Device types** — character devices (sequential byte streams: terminals, serial ports, `/dev/null`) vs. block devices (random-access blocks: disks, partitions). Identified by major/minor number pairs.
- **Device nodes** — special files in `/dev` created by `mknod`. The major number identifies the driver; the minor number identifies the specific device instance.
- **Key devices**: `/dev/null` (discard sink), `/dev/zero` (zero-byte source), `/dev/full` (always-full device), `/dev/random` and `/dev/urandom` (entropy sources — blocking vs. non-blocking), `/dev/loop*` (loopback block devices).
- **ioctl** — the catch-all interface for device-specific operations that don't fit the read/write/seek model. Takes a file descriptor, a request code, and an optional argument. Examples: querying terminal size (`TIOCGWINSZ`), ejecting CD-ROMs.
- **PTYs** — pseudo-terminals: master/slave pairs for terminal emulators and remote shells. `/dev/ptmx` allocates a new PTY pair.

See [devices](../entities/devices.md).

### Ch 7: The /proc File System

The kernel's primary interface for exposing runtime information to user space:

- **Process entries** (`/proc/<pid>/`) — per-process virtual files:
  - `status` — process state, memory usage, UIDs, signal masks.
  - `cmdline` — command line (null-separated argv).
  - `environ` — environment variables.
  - `maps` — memory-mapped regions (VMAs) with permissions and backing files.
  - `fd/` — directory of symlinks to open file descriptors.
  - `stat`, `statm` — raw numeric process/memory statistics.
  - `exe` — symlink to the executable.
  - `cwd`, `root` — symlinks to working directory and root directory.
- **Hardware info** — `/proc/cpuinfo` (CPU model, features, speed), `/proc/pci` or `/proc/bus/pci` (PCI devices), `/proc/interrupts` (IRQ assignments and counts), `/proc/dma`, `/proc/ioports`, `/proc/iomem`.
- **Kernel info** — `/proc/version` (kernel version string), `/proc/sys/` (tunable kernel parameters, writable to change behavior at runtime), `/proc/modules` (loaded modules).
- **System statistics** — `/proc/uptime`, `/proc/loadavg`, `/proc/meminfo` (physical/swap/buffer memory breakdown), `/proc/vmstat`, `/proc/stat` (CPU time breakdown, context switches, boot time).
- **Self link** — `/proc/self` is a symlink to the current process's `/proc/<pid>` directory, useful for introspection without knowing your own PID.

See [proc-filesystem](../entities/proc-filesystem.md).

### Ch 8: Linux System Calls

A reference chapter covering individual system calls not covered elsewhere, with usage patterns and examples:

| System call | Purpose |
|-------------|---------|
| `strace` | Trace system calls made by a process (diagnostic tool, not a syscall itself) |
| `access()` | Check file permissions without opening — test `R_OK`, `W_OK`, `X_OK`, `F_OK` |
| `fcntl()` | File descriptor manipulation — duplicate FDs, get/set flags (`O_NONBLOCK`, `FD_CLOEXEC`), advisory record locking |
| `fsync()` / `fdatasync()` | Flush file data/metadata to disk — `fsync()` flushes both, `fdatasync()` only data |
| `getrlimit()` / `setrlimit()` | Query/set resource limits: `RLIMIT_CPU`, `RLIMIT_FSIZE`, `RLIMIT_NOFILE`, `RLIMIT_NPROC`, etc. |
| `getrusage()` | Get resource usage statistics: user/system CPU time, page faults, context switches |
| `gettimeofday()` | Microsecond-precision wall-clock time |
| `mlock()` / `mlockall()` | Lock pages in physical memory (prevent swapping) — requires `CAP_IPC_LOCK` |
| `mmap()` | Map files or anonymous memory into the address space — used for file I/O, IPC, and allocators |
| `nanosleep()` | High-resolution sleep (nanosecond granularity, though actual resolution is system-dependent) |
| `readlink()` | Read the target of a symbolic link without following it |
| `sendfile()` | Zero-copy file-to-socket transfer — the kernel moves data directly without user-space buffering |
| `setitimer()` | Set interval timers: `ITIMER_REAL` (wall clock → `SIGALRM`), `ITIMER_VIRTUAL` (user CPU → `SIGVTALRM`), `ITIMER_PROF` (user+system CPU → `SIGPROF`) |
| `sysinfo()` | System-wide statistics: uptime, load averages, total/free RAM and swap |
| `uname()` | System identification: OS name, hostname, kernel release/version, machine architecture |

See [system-calls](../entities/system-calls.md).

### Ch 9: Inline Assembly Code

Writing inline assembly in GCC for performance-critical or hardware-specific code:

- **GAS (GNU Assembler) syntax** — AT&T style: `mnemonic source, dest`, register names prefixed with `%`, immediate values with `$`, memory operands with displacement/base/index syntax.
- **The `asm` construct** — `asm("instructions" : outputs : inputs : clobbers)`. Extended asm allows the compiler to manage register allocation via constraints.
- **Constraints** — `"r"` (any register), `"m"` (memory), `"i"` (immediate), `"a"`/`"b"`/`"c"`/`"d"` (specific x86 registers). `"="` for output-only, `"+"` for read-write.
- **Volatile asm** — `asm volatile(...)` prevents the compiler from optimizing away or reordering the assembly block.
- **Use cases** — accessing CPU special registers (e.g., `rdtsc` for cycle counting), atomic operations, CPUID queries, I/O port access.

See [concept-inline-assembly](../concepts/concept-inline-assembly.md).

### Ch 10: Security

Linux security model from the application developer's perspective:

- **Users and groups** — UIDs, GIDs, supplementary groups. `/etc/passwd`, `/etc/group`. The superuser (UID 0) bypasses most permission checks.
- **Process credentials** — real UID/GID (who you are), effective UID/GID (what permissions you have), saved set-UID (for toggling privilege). `setuid()`/`seteuid()`/`setreuid()`.
- **Set-UID/Set-GID programs** — executables that run with the file owner's/group's effective ID. The `chmod u+s` / `chmod g+s` mechanism. Security implications and best practices (drop privileges early, minimize privileged code).
- **The sticky bit** — on directories, prevents users from deleting files they don't own (used on `/tmp`).
- **File permissions** — the `rwx` model for owner/group/other, `umask`, `chmod`, `chown`.
- **Capabilities** — fine-grained decomposition of root privilege into individual capabilities (`CAP_NET_BIND_SERVICE`, `CAP_SYS_ADMIN`, `CAP_DAC_OVERRIDE`, etc.). Allows granting specific privileges without full root.
- **chroot jails** — `chroot()` changes a process's view of the filesystem root. Used for sandboxing, but not a complete security boundary (root can escape).
- **PAM (Pluggable Authentication Modules)** — framework for abstracting authentication: `/etc/pam.d/` configuration, module stacking, conversation functions. Allows programs to authenticate users without hardcoding a specific mechanism.

See [concept-linux-security](../concepts/concept-linux-security.md).

### Ch 11: A Sample GNU/Linux Application

Demonstrates building a complete server application that ties together all the book's concepts:

- **Architecture** — a multi-process HTTP server. The main process listens for connections and forks a child to handle each request.
- **Implementation patterns used**:
  - Socket programming: `socket()`, `bind()`, `listen()`, `accept()` loop.
  - Process-per-connection concurrency model (vs. threading or event-driven).
  - `SIGCHLD` handling for child reaping.
  - CGI-like module system using `fork()`+`exec()` and `dup2()` for I/O redirection.
  - `/proc` filesystem queries as dynamic content generators.
  - Shared memory for inter-module state.
  - Signal-based graceful shutdown.
- **Design lessons** — separation of server framework from content modules, graceful error handling, resource cleanup.

### Appendixes

- **Appendix A: Other Development Tools** — `gprof` (profiling), `gcov` (code coverage), `dmalloc` (memory debugging), `mtrace` (malloc tracing), `valgrind` (memory error detection), `objdump` and `nm` (binary inspection), `strace` and `ltrace` (system/library call tracing), `Electric Fence` (buffer overflow detection).
- **Appendix B: Low-Level I/O** — `open()`/`close()`/`read()`/`write()`/`lseek()` — the raw file descriptor interface beneath `<stdio.h>`. `O_CREAT`, `O_TRUNC`, `O_APPEND`, `O_NONBLOCK` flags. `stat()`/`fstat()` for file metadata. Directory operations: `opendir()`/`readdir()`/`closedir()`.
- **Appendix C: Table of Signals** — Complete signal reference with numbers, default actions, and descriptions for all standard Linux signals (SIGHUP through SIGSYS, plus real-time signals).
- **Appendix D: Online Resources** — Pointers to kernel.org, GNU project, LDP, man pages, GCC docs.
- **Appendix E: Open Publication License** — The license under which the book was released.
- **Appendix F: GNU General Public License** — Full GPL v2 text, included because the book contains GPL-licensed code examples.

## Key Themes

1. **User-space perspective** — Every topic is covered from the application developer's viewpoint, showing how to use kernel features correctly rather than explaining how they are implemented internally.
2. **Defensive programming** — The book consistently emphasizes error checking, resource cleanup, signal safety, and avoiding common pitfalls (race conditions, zombie processes, TOCTOU bugs).
3. **Unix philosophy** — Programs should compose via pipes, use standard I/O conventions, handle signals gracefully, and follow the "do one thing well" principle.
4. **Practical concurrency** — Detailed coverage of both the multi-process model (fork/exec) and multi-threaded model (pthreads), with guidance on when to use each.
5. **Security awareness** — Privilege management, input validation, and the principle of least privilege are woven throughout rather than isolated in one chapter.

## See also

- [process-management](../entities/process-management.md)
- [pthreads](../entities/pthreads.md)
- [ipc](../entities/ipc.md)
- [signals](../entities/signals.md)
- [system-calls](../entities/system-calls.md)
- [proc-filesystem](../entities/proc-filesystem.md)
- [devices](../entities/devices.md)
- [concept-linux-security](../concepts/concept-linux-security.md)
- [concept-inline-assembly](../concepts/concept-inline-assembly.md)
