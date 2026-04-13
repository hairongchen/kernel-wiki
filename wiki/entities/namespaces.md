---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [professional-linux-kernel-architecture]
tags: [namespaces, containers, isolation, pid-namespace, nsproxy]
---

# Linux Kernel Namespaces

## Overview

Namespaces provide lightweight virtualization by isolating global system resources so
that each group of processes sees its own independent instance of those resources.
They are the kernel-level building blocks for container technologies such as LXC and
Docker. Rather than being introduced all at once, namespace support was added
incrementally across kernel versions. By Linux 2.6.24, the kernel includes six
namespace types: PID, UTS, IPC, mount, user, and network namespaces.

Each process belongs to exactly one instance of every namespace type. By default, a
child process inherits all namespaces from its parent. The `clone()` system call
accepts `CLONE_NEW*` flags to place the child into newly created namespaces, while
`unshare()` allows a process to create new namespaces for itself without forking.

## struct nsproxy

The kernel groups all namespace pointers for a task into a single structure,
`struct nsproxy`, stored in `task_struct->nsproxy`. Its key fields are:

| Field      | Type                        | Description                              |
|------------|-----------------------------|------------------------------------------|
| `count`    | `atomic_t`                  | Reference count (shared across tasks)    |
| `uts_ns`   | `struct uts_namespace *`    | UTS namespace (hostname, domain)         |
| `ipc_ns`   | `struct ipc_namespace *`    | IPC namespace (System V IPC)             |
| `mnt_ns`   | `struct mnt_namespace *`    | Mount namespace (filesystem hierarchy)   |
| `pid_ns`   | `struct pid_namespace *`    | PID namespace (process IDs)              |
| `net_ns`   | `struct net *`              | Network namespace (network stack)        |

Multiple tasks that share all their namespaces point to the same `nsproxy` instance,
and the reference count tracks how many tasks hold a reference. When a process calls
`clone()`, the kernel invokes `copy_namespaces()`, which inspects the `CLONE_NEW*`
flags. If any flag is set, a new `nsproxy` is allocated and only the corresponding
namespace pointer(s) are replaced with freshly created namespace instances; the
remaining pointers are shared with the parent.

## PID Namespace

The PID namespace is the most complex namespace type. It virtualizes process IDs so
that processes in different PID namespaces can have the same PID number without
conflict. PID namespaces are hierarchical: a parent namespace can observe all
processes in its child namespaces, but a child namespace cannot see processes in the
parent.

### struct pid_namespace

Each PID namespace is represented by `struct pid_namespace`, which contains:

- **`pidmap`** -- a bitmap used for allocating PID numbers within the namespace.
- **`level`** -- the nesting depth of this namespace (the root namespace has level 0).
- **`child_reaper`** -- a pointer to the init process (PID 1) for this namespace.
  This process adopts and reaps orphaned children, just as the global init does in
  the root namespace.

### struct pid and struct upid

The kernel uses `struct pid` as its internal representation of a process identifier.
Because a process is visible in its own PID namespace and every ancestor namespace,
a single `struct pid` may correspond to multiple numeric PID values. It contains a
`numbers[]` array of `struct upid` entries, one per namespace level in which the
process is visible.

`struct upid` is a simple pair:

- **`nr`** -- the numeric PID value within a specific namespace.
- **`ns`** -- a pointer to the `struct pid_namespace` where `nr` is valid.

For example, a process created in a level-2 PID namespace has three `upid` entries:
one for the level-2 namespace, one for the level-1 namespace, and one for the root
(level-0) namespace. Each entry holds a different numeric PID.

### PID Translation

The kernel provides namespace-aware helpers for PID translation:

- **`task_pid_nr(task)`** -- returns the PID of `task` as seen in the caller's
  current PID namespace.
- **`task_pid_nr_ns(task, ns)`** -- returns the PID of `task` within a specific
  namespace `ns`.

These functions walk the `numbers[]` array to find the `upid` entry matching the
requested namespace.

### Creating a PID Namespace

Calling `clone()` with the `CLONE_NEWPID` flag creates a new PID namespace. The
child process becomes PID 1 in the new namespace and serves as the child reaper for
that namespace. The child is also assigned a PID in every ancestor namespace. If the
namespace-local init (PID 1) exits, all remaining processes in that namespace are
terminated.

## UTS Namespace

The UTS namespace isolates the hostname and domain name returned by `uname()`. It is
represented by `struct uts_namespace`, which contains a `struct new_utsname` with
fields including `nodename` and `domainname`.

When a process calls `uname()`, the kernel reads the values from the UTS namespace
attached to the calling task's `nsproxy`. Creating a new UTS namespace via `clone()`
with `CLONE_NEWUTS` gives the child a private copy of these identifiers, which can
be changed independently without affecting other namespaces.

## IPC Namespace

The IPC namespace isolates System V IPC objects: semaphore sets, message queues, and
shared memory segments. Each IPC namespace, represented by `struct ipc_namespace`,
maintains its own set of IPC ID arrays. Processes in one IPC namespace cannot see or
access IPC objects belonging to another.

A new IPC namespace is created with the `CLONE_NEWIPC` flag during `clone()`. The
child starts with an empty set of IPC objects.

## Mount Namespace

The mount namespace isolates the filesystem mount table. Each mount namespace
maintains its own tree of `struct vfsmount` entries, so processes in different mount
namespaces see entirely different filesystem hierarchies. A process can mount or
unmount filesystems in its namespace without affecting processes in other namespaces.

Mount namespaces were the first namespace type added to the kernel, which is why
their flag is named `CLONE_NEWNS` (simply "new namespace") rather than following the
`CLONE_NEW<type>` convention used by later namespace types.

## Network Namespace

The network namespace isolates the entire network stack. Each network namespace,
represented by `struct net`, contains its own:

- **Loopback device** -- every network namespace has an independent `lo` interface.
- **Routing tables** -- separate FIB and routing cache instances.
- **Netfilter/iptables rules** -- independent packet filtering configuration.
- **`/proc/net`** -- each namespace presents its own view of network statistics.
- **Sockets** -- sockets are bound to their creating namespace.

A new network namespace is created with the `CLONE_NEWNET` flag. Initially the
namespace contains only the loopback device; additional virtual or physical devices
must be assigned to it explicitly.

## User Namespace

The user namespace isolates user and group IDs. A process can have UID 0 (root)
inside its user namespace while being an unprivileged user in the parent namespace.
This enables unprivileged containers where processes believe they have full root
capabilities but are actually constrained.

A new user namespace is created with the `CLONE_NEWUSER` flag. In Linux 2.6.24 the
implementation is minimal -- full support for UID/GID mapping, nested user
namespaces, and capability isolation was developed in later kernel versions
(primarily 3.8 and beyond).

## Creating Namespaces

There are two primary interfaces for creating namespaces:

1. **`clone()`** -- the `CLONE_NEW*` flags instruct the kernel to create new
   namespace instances for the child process. Multiple flags can be combined in a
   single `clone()` call to create several new namespaces at once.

2. **`unshare()`** -- creates new namespaces for the calling process without
   forking. The process leaves its current namespace and enters a newly created one.
   This is useful when a process wants to isolate itself after it has already
   started.

The `CLONE_NEW*` flags and their corresponding namespace types:

| Flag             | Namespace |
|------------------|-----------|
| `CLONE_NEWPID`   | PID       |
| `CLONE_NEWUTS`   | UTS       |
| `CLONE_NEWIPC`   | IPC       |
| `CLONE_NEWNS`    | Mount     |
| `CLONE_NEWNET`   | Network   |
| `CLONE_NEWUSER`  | User      |

## Namespace-Aware APIs

Throughout the kernel, subsystem code has been updated to operate in a
namespace-aware manner. Rather than accessing global resource tables directly,
kernel functions consult the calling task's namespace. Examples include:

- **`find_task_by_vpid()`** -- looks up a task by its virtual PID in the caller's
  current PID namespace, rather than using a global PID.
- **`ip_route_output_key()`** -- performs route lookups using the routing tables
  from the task's [network namespace](routing-subsystem.md).
- **`ipc_findkey()`** -- searches for IPC objects within the task's IPC namespace.

This pervasive namespace awareness ensures that processes within a namespace
interact only with resources visible in that namespace, providing the isolation
guarantees that containers depend on.

## See also

- [process-management](process-management.md)
- [ipc](ipc.md)
- [routing-subsystem](routing-subsystem.md)
- [concept-linux-security](../concepts/concept-linux-security.md)
