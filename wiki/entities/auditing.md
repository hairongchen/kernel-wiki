---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [professional-linux-kernel-architecture]
tags: [auditing, security, netlink, selinux, syscall-tracing]
---

# Linux Kernel Auditing Subsystem

## Overview

The Linux audit subsystem provides detailed logging of security-relevant events, with system call execution being the primary focus. It serves several key purposes:

- **Compliance**: Supports CAPP (Controlled Access Protection Profile) and EAL (Evaluation Assurance Level) certification requirements.
- **Intrusion detection**: Captures granular records of system activity that security tools can analyze for suspicious behavior.
- **Forensic analysis**: Provides a tamper-evident trail of kernel-level events for post-incident investigation.

The subsystem follows a split architecture: the kernel captures events and applies filtering rules, while the user-space daemon `auditd` receives these events via a [netlink](net-device.md) socket and writes them to persistent storage. This design keeps the kernel-side component lightweight while delegating policy and storage decisions to user space.

## Architecture

### Kernel-Side Components

The audit subsystem hooks into two primary locations within the kernel:

1. **System call entry and exit points** — Every [system call](system-calls.md) passes through audit checkpoints that can capture the syscall number, arguments, and return value.
2. **Security module hooks** — SELinux's Access Vector Cache (AVC) emits audit records when access decisions are logged, providing visibility into mandatory access control enforcement.

### The `audit_context` Structure

Each task in the kernel can carry an audit context, attached via `task_struct->audit_context`. The `struct audit_context` is the central per-task data structure for the audit subsystem and contains:

- **Audit state** — One of three values:
  - `AUDIT_DISABLED` — No auditing for this task
  - `AUDIT_BUILD_CONTEXT` — Collecting information but not yet committed to emitting a record
  - `AUDIT_RECORD_CONTEXT` — A full audit record will be generated at syscall exit
- **`in_syscall` flag** — Indicates whether the task is currently inside a system call
- **Syscall metadata** — Syscall number, architecture, and return value
- **Argument capture fields** — Storage for system call arguments that matched audit rules
- **Name and inode information** — Filesystem-related data captured during path resolution (filenames, inodes, device numbers) for operations that touch the filesystem

### User-Space Communication

Communication between the kernel and `auditd` uses the `NETLINK_AUDIT` socket type. This is a dedicated netlink protocol family that provides bidirectional messaging:

- **Kernel to user space**: Audit event records flow from the kernel to `auditd`.
- **User space to kernel**: `auditd` (and `auditctl`) send commands to configure the audit subsystem.

## Netlink Interface

The function `audit_receive_msg()` is the kernel-side handler for commands arriving from user space over the `NETLINK_AUDIT` socket. It processes the following message types:

| Message Type | Purpose |
|---|---|
| `AUDIT_SET` | Enable or disable auditing globally, set backlog limit, configure failure behavior |
| `AUDIT_GET` | Query current audit status (enabled state, PID of auditd, backlog stats) |
| `AUDIT_LIST_RULES` | List all currently loaded audit rules |
| `AUDIT_ADD_RULE` | Add a new audit rule to a filter list |
| `AUDIT_DEL_RULE` | Remove an existing audit rule from a filter list |

When the kernel needs to send an audit event to user space, it calls `audit_log_end()`, which in turn calls `netlink_unicast()` to deliver the record directly to the registered `auditd` process.

## Audit Rules

### Filter Lists

Audit rules are organized into filter lists, each evaluated at a different point in execution:

- **`AUDIT_FILTER_ENTRY`** — Evaluated at system call entry. Determines whether to begin building an audit context for this syscall.
- **`AUDIT_FILTER_EXIT`** — Evaluated at system call exit. Can trigger audit record generation based on the syscall's outcome (return value, side effects).
- **`AUDIT_FILTER_TASK`** — Evaluated at task creation. Applies audit policy to newly forked processes.
- **`AUDIT_FILTER_USER`** — Evaluated for user-space-originated audit messages.

### Rule Structures

Each audit rule in the kernel is represented by `struct audit_krule`, which contains:

- **`listnr`** — Which filter list this rule belongs to (entry, exit, task, or user)
- **`action`** — What to do when the rule matches:
  - `AUDIT_ALWAYS` — Generate an audit record
  - `AUDIT_NEVER` — Suppress audit record generation
- **`field_count`** — Number of match criteria in this rule
- **`fields[]`** — Array of `struct audit_field` entries defining match criteria

Each `struct audit_field` specifies a single match criterion:

- **`type`** — The attribute to match against (e.g., `AUDIT_PID`, `AUDIT_UID`, `AUDIT_ARCH`, `AUDIT_SYSCALL`)
- **`op`** — Comparison operator (equal, not equal, greater than, less than)
- **`val`** — The value to compare against

Rules are evaluated at syscall entry by `audit_filter_syscall()`, which walks the appropriate filter list and checks each rule's fields against the current task and syscall context.

## System Call Auditing Flow

The audit subsystem intercepts system calls at three stages:

### 1. Syscall Entry — `audit_syscall_entry()`

Called as the task enters kernel mode for a system call. This function:

- Records the syscall number and architecture (important for multi-arch systems where 32-bit and 64-bit syscall numbers differ)
- Captures the raw system call arguments
- Evaluates the entry filter rules to determine whether an audit context should be built for this syscall

### 2. During Syscall Execution

As the kernel processes the system call, various audit hooks capture additional context that enriches the final audit record:

- **`audit_getname()`** — Records filenames passed to filesystem operations (open, unlink, rename, etc.)
- **`audit_inode()`** — Captures inode numbers and device information for accessed filesystem objects

This incremental context collection is why `AUDIT_BUILD_CONTEXT` exists as a state — the kernel accumulates data throughout the syscall before deciding whether to emit a record.

### 3. Syscall Exit — `audit_syscall_exit()`

Called as the task returns from the system call. This function:

- Records the syscall return value
- Evaluates the exit filter rules (which can match on return values and accumulated context)
- If auditing is enabled for this syscall (state is `AUDIT_RECORD_CONTEXT`), calls `audit_log_exit()` to format and emit the complete audit record
- Resets the audit context for the next syscall

## SELinux AVC Auditing

SELinux's Access Vector Cache generates audit records when access control decisions are logged. The function `avc_audit()` is responsible for creating these records, which are emitted independently of syscall auditing.

An AVC audit record includes:

- **Source security context** — The SELinux label of the subject (process) requesting access
- **Target security context** — The SELinux label of the object being accessed
- **Permission class** — The object class (file, socket, process, etc.)
- **Specific permissions** — The individual permissions checked (read, write, execute, etc.)
- **Decision** — Whether access was denied or allowed (allowed accesses are only logged when explicitly configured)

These records complement syscall audit records by providing the mandatory access control dimension of security events.

## Audit Record Format

Each audit record emitted by the kernel contains:

- **Type** — Identifies the record kind: `AUDIT_SYSCALL`, `AUDIT_PATH`, `AUDIT_CWD`, `AUDIT_AVC`, among others
- **Timestamp** — When the event occurred
- **Serial number** — A unique identifier for the event

The body of each record consists of key-value pairs specific to the record type (e.g., `syscall=2 arch=c000003e success=yes exit=3 pid=1234 uid=0`).

A single auditable event often produces multiple records — for example, a file open might generate an `AUDIT_SYSCALL` record, an `AUDIT_CWD` record, and one or more `AUDIT_PATH` records. Records that share the same timestamp and serial number are grouped together to form a single logical audit event.

## See also

- [concept-linux-security](../concepts/concept-linux-security.md)
- [system-calls](system-calls.md)
- [proc-filesystem](proc-filesystem.md)
