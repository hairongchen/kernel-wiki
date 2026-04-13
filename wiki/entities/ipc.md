---
type: entity
created: 2026-04-08
updated: 2026-04-09
sources: [understanding-the-linux-kernel, advanced-linux-programming]
tags: [ipc, pipes, shared-memory, semaphores, message-queues, sockets]
---

# Inter-Process Communication (IPC)

Linux provides multiple IPC mechanisms, ranging from the simple (pipes) to the feature-rich (System V shared memory). Each offers different trade-offs in terms of data copying, synchronization, persistence, and namespace isolation.

## Pipes

A **pipe** is a unidirectional, byte-stream communication channel between related processes (typically parent and child). Pipes are the oldest Unix IPC mechanism and the backbone of shell pipelines (`cmd1 | cmd2`).

### Implementation

A pipe is represented by a `pipe_inode_info` structure associated with an inode in the kernel's special **pipefs** filesystem (a virtual filesystem that exists only in memory, never on disk):

```c
struct pipe_inode_info {
    wait_queue_head_t wait;      /* readers/writers sleep here */
    char *base;                   /* buffer base address */
    unsigned int readers;         /* number of read file descriptors */
    unsigned int writers;         /* number of write file descriptors */
    unsigned int r_counter;       /* reader reference count */
    unsigned int w_counter;       /* writer reference count */
    /* ... */
};
```

The pipe buffer consists of **16 page-sized buffers** (16 x 4 KB = 64 KB total), organized as a circular buffer. Reads and writes operate on this circular buffer with the following semantics:

| Operation | Buffer state | Behavior |
|-----------|-------------|----------|
| **Read** | Non-empty | Copies available data to user buffer, advances read position |
| **Read** | Empty, writers exist | Blocks (sleeps on `wait`) until data arrives |
| **Read** | Empty, no writers | Returns 0 (EOF) |
| **Write** | Not full | Copies data from user buffer, advances write position |
| **Write** | Full, readers exist | Blocks until space is available |
| **Write** | Full or not, no readers | `SIGPIPE` sent to writer, returns `-EPIPE` |

Writes of `PIPE_BUF` bytes (4096 on Linux) or less are guaranteed **atomic** — they will not be interleaved with writes from other processes. Larger writes may be broken up.

### pipe() System Call

`pipe()` creates two file descriptors: `fd[0]` for reading and `fd[1]` for writing. Both reference the same underlying `pipe_inode_info`. After `fork()`, the parent typically closes one end and the child closes the other, establishing a unidirectional channel.

The pipe's file operations are defined by `read_pipe_fops` and `write_pipe_fops`, which implement the blocking/waking semantics and circular buffer management.

## FIFOs (Named Pipes)

A **FIFO** is a pipe that has a name in the filesystem, created with `mkfifo()` or `mknod()` with the `S_IFIFO` type. Unlike anonymous pipes, FIFOs allow communication between **unrelated processes** — any process that opens the FIFO's pathname can read or write.

FIFOs use the same `pipe_inode_info` implementation as anonymous pipes. The key differences are:

- FIFOs have a persistent filesystem name (visible via `ls -l`).
- Opening a FIFO for read blocks until a writer opens it (and vice versa), unless `O_NONBLOCK` is used.
- Multiple readers or multiple writers can open the same FIFO concurrently.

## System V IPC

System V IPC is a set of three related mechanisms — semaphores, message queues, and shared memory — that share a common infrastructure for identification, permissions, and lifecycle management.

### Common Infrastructure

All System V IPC objects share:

- **Key**: a 32-bit value (`key_t`) used by unrelated processes to find the same object. The special value `IPC_PRIVATE` (0) creates a private object accessible only via the returned ID.
- **ID**: a kernel-assigned integer identifier returned by `semget()`, `msgget()`, or `shmget()`. Used for all subsequent operations.
- **Permissions**: an `ipc_perm` structure containing owner UID/GID, creator UID/GID, and a Unix-style mode (read/write for owner, group, others).
- **ipc_ids**: the kernel maintains an `ipc_ids` structure for each IPC type (semaphores, messages, shared memory). This structure uses a **radix tree** (in place of the older array) to map IPC IDs to kernel objects, allowing fast lookup and supporting a large number of objects.

### IPC Namespaces

Linux 2.6 introduces **IPC namespaces** for container isolation. Each namespace has its own independent set of `ipc_ids` for semaphores, message queues, and shared memory. Processes in different IPC namespaces cannot see or interact with each other's IPC objects. This is a key building block for lightweight virtualization (containers).

### System V Semaphores

System V semaphores are counting semaphores that support **atomic multi-semaphore operations** — a single `semop()` call can atomically increment, decrement, or wait-for-zero on multiple semaphores in a set. This capability distinguishes them from POSIX semaphores and kernel semaphores (see [concept-kernel-synchronization](../concepts/concept-kernel-synchronization.md)).

#### Data Structures

```
shmid_kernel -> sem_array -> sem[0..nsems-1]
                          -> sem_queue (pending operations)
                          -> sem_undo (undo entries)
```

- `sem_array`: represents a semaphore set. Contains an array of `struct sem` (each with a `semval` counter and a pending list), a global pending list (`sem_pending`), and undo list.
- `sem_queue`: each pending `semop()` that cannot complete immediately is represented by a `sem_queue` entry linked to the semaphore set.

#### Key Operations

| Syscall | Purpose |
|---------|---------|
| `semget(key, nsems, flags)` | Create or find a semaphore set |
| `semop(semid, sops, nsops)` | Perform an array of operations atomically. Each `sembuf` specifies: semaphore index, operation value (positive = release, negative = acquire, zero = wait-for-zero), and flags (`IPC_NOWAIT`, `SEM_UNDO`) |
| `semctl(semid, semnum, cmd, ...)` | Control operations: `SETVAL`, `GETVAL`, `IPC_RMID` (destroy), `IPC_STAT`, etc. |

#### SEM_UNDO Mechanism

If a process requests `SEM_UNDO` on a `semop()`, the kernel records an **undo entry** that will automatically reverse the operation if the process exits without explicitly undoing it. This prevents a crashed process from leaving semaphores in an inconsistent state. The undo entries are stored in per-process `sem_undo` structures linked from both the `task_struct` and the `sem_array`.

### System V Message Queues

Message queues allow processes to exchange **typed messages** — each message has a `long` type field and a variable-length data payload. Receivers can selectively retrieve messages by type, implementing priority or category-based filtering.

#### Data Structures

- `msg_queue`: the kernel object for a message queue. Contains a linked list of `msg_msg` structures, current size tracking, and maximum size limits.
- `msg_msg`: represents a single message. Contains the `mtype` (type), `mtext` (data), and size. For messages larger than one page, the data is stored in a linked list of `msg_msgseg` structures.

#### Key Operations

| Syscall | Purpose |
|---------|---------|
| `msgget(key, flags)` | Create or find a message queue |
| `msgsnd(msqid, msgp, msgsz, flags)` | Send a message. Blocks if the queue is full (unless `IPC_NOWAIT`). The message is copied from user space into kernel-allocated `msg_msg` structures. |
| `msgrcv(msqid, msgp, msgsz, msgtyp, flags)` | Receive a message. `msgtyp` controls selection: 0 = first message, positive = first message of that type, negative = first message with type <= |msgtyp| (lowest type first). Blocks if no matching message (unless `IPC_NOWAIT`). |
| `msgctl(msqid, cmd, buf)` | Control: `IPC_RMID`, `IPC_STAT`, `IPC_SET` |

### System V Shared Memory

Shared memory is the fastest IPC mechanism — once established, processes read and write a shared memory region with no kernel involvement (no system calls, no data copying). Synchronization must be provided separately (e.g., via semaphores or futexes).

#### Data Structures

- `shmid_kernel`: the kernel object for a shared memory segment. Contains a pointer to the backing `address_space` (file-based, using the kernel's `shmfs` / `tmpfs`), size, permissions, and attachment count.

#### Key Operations

| Syscall | Purpose |
|---------|---------|
| `shmget(key, size, flags)` | Create or find a shared memory segment |
| `shmat(shmid, shmaddr, flags)` | Attach the segment to the calling process's address space. Internally calls `do_mmap()` to create a VMA backed by the segment's `tmpfs` file. Returns the virtual address. `shmaddr` can suggest an address (or NULL for kernel-chosen). `SHM_RDONLY` flag creates a read-only mapping. |
| `shmdt(shmaddr)` | Detach the segment (calls `do_munmap()`) |
| `shmctl(shmid, cmd, buf)` | Control: `IPC_RMID` (mark for deletion — actual deletion deferred until all attachments are removed), `IPC_STAT`, `SHM_LOCK` (lock pages in RAM) |

The key insight is that `shmat()` maps the shared memory segment into the process's virtual address space exactly like `mmap()` maps a file. The shared memory segment is backed by a file in `tmpfs` (a RAM-based filesystem), so it persists across `shmat()`/`shmdt()` cycles but is lost on reboot.

## POSIX Message Queues

POSIX message queues are a more modern alternative to System V message queues, providing:

- **Priority-based ordering**: each message has an integer priority, and `mq_receive()` always returns the highest-priority message first (within the same priority, FIFO order).
- **Asynchronous notification**: `mq_notify()` registers for notification (via signal or thread creation) when a message arrives on a previously empty queue.
- **Filesystem interface**: POSIX message queues are implemented via the **mqueue filesystem**. Each queue appears as a file, and standard file permissions apply. The filesystem is typically mounted at `/dev/mqueue`.

### Key API

| Function | Purpose |
|----------|---------|
| `mq_open(name, oflag, mode, attr)` | Create or open a message queue |
| `mq_send(mqd, msg_ptr, msg_len, priority)` | Send a message with a given priority |
| `mq_receive(mqd, msg_ptr, msg_len, *priority)` | Receive the highest-priority message |
| `mq_notify(mqd, sevp)` | Register for arrival notification |
| `mq_close(mqd)` / `mq_unlink(name)` | Close descriptor / remove queue |
| `mq_getattr()` / `mq_setattr()` | Query/set queue attributes (max messages, max message size, current count) |

The `mq_timedreceive()` and `mq_timedsend()` variants accept an absolute timeout, unlike System V's binary blocking/non-blocking model.

## Memory-Mapped Files as IPC

Beyond System V shared memory, `mmap()` with `MAP_SHARED` on a regular file provides an alternative shared-memory mechanism. Multiple processes mapping the same file with `MAP_SHARED` see each other's writes directly. `msync()` flushes changes to the backing file for durability. This approach avoids the System V IPC API complexity and leverages the [page-cache](page-cache.md) for coherency.

## UNIX Domain Sockets

UNIX domain sockets (`AF_UNIX` / `PF_UNIX`) provide bidirectional IPC through the socket API. Unlike network sockets, they use filesystem pathnames (or abstract namespace) for addressing and communicate entirely within the kernel — no network stack overhead.

| Mode | API | Characteristics |
|------|-----|-----------------|
| **Stream** (`SOCK_STREAM`) | `connect()` / `accept()` | Reliable, ordered, connection-oriented byte stream (like TCP but local) |
| **Datagram** (`SOCK_DGRAM`) | `sendto()` / `recvfrom()` | Unreliable (but on local machine, effectively reliable), message-oriented |

`socketpair()` creates a pre-connected pair of UNIX domain sockets, useful between parent and child processes (similar to `pipe()` but bidirectional).

UNIX domain sockets are used extensively for local client-server communication: X11, D-Bus, syslog, Docker, systemd, and many database servers use them as their primary local transport.

### Ancillary Data

UNIX domain sockets support passing **ancillary data** via `sendmsg()`/`recvmsg()`:
- **File descriptor passing** (`SCM_RIGHTS`) — send open file descriptors to another process, even an unrelated one. The kernel duplicates the FD into the receiver's table.
- **Credential passing** (`SCM_CREDENTIALS`) — send/verify the sender's PID, UID, and GID (kernel-verified, cannot be forged).

## IPC Mechanism Comparison

| Mechanism | Direction | Related procs? | Data copying | Synchronization | Persistence |
|-----------|-----------|---------------|--------------|-----------------|-------------|
| Pipe | Unidirectional | Yes (parent-child) | Kernel buffer | Built-in (blocking) | Process lifetime |
| FIFO | Unidirectional | No | Kernel buffer | Built-in (blocking) | Filesystem name persists |
| Shared memory | Bidirectional | No | None (after setup) | Must provide separately | Until `IPC_RMID` + detach |
| mmap shared file | Bidirectional | No | None (after setup) | Must provide separately | File persists |
| Message queue | Bidirectional | No | Kernel copy | Built-in (blocking) | Until `IPC_RMID` |
| UNIX socket | Bidirectional | No | Kernel copy | Built-in (blocking) | Filesystem name persists |

## See also

- [process-management](process-management.md)
- [pthreads](pthreads.md)
- [concept-kernel-synchronization](../concepts/concept-kernel-synchronization.md)
- [program-execution](program-execution.md)
- [devices](devices.md)
