---
type: concept
created: 2026-04-09
updated: 2026-04-09
sources: [advanced-linux-programming]
tags: [security, permissions, capabilities, setuid, chroot, pam]
---

# Linux Security Model

Linux implements a multi-layered security model rooted in the traditional Unix discretionary access control (DAC) system, extended with POSIX capabilities for fine-grained privilege management. This page covers the security mechanisms from the application developer's perspective — what a programmer needs to understand to write secure software on Linux.

## Users, Groups, and the Superuser

Every process runs with a set of credentials that determine its access rights:

- **UID (User ID)** — the numeric identity of the user. UID 0 is the **superuser** (root), which bypasses most permission checks.
- **GID (Group ID)** — the primary group of the process.
- **Supplementary groups** — additional group memberships (up to 32 by default, configurable).

User-to-UID and group-to-GID mappings are stored in `/etc/passwd` and `/etc/group` (or network directory services like LDAP/NIS).

## Process Credentials

Each process carries three sets of UID/GID values:

| Credential | Purpose | Set by |
|------------|---------|--------|
| **Real UID/GID** | Identifies who actually started the process | Inherited from parent; only root can change arbitrarily |
| **Effective UID/GID** | Used for permission checks (file access, IPC, signals) | Set-UID/set-GID execution, or explicit `seteuid()` / `setegid()` |
| **Saved set-UID/GID** | Preserves the old effective ID for later restoration | Set by the kernel on exec of a set-UID binary |

This three-level scheme enables **privilege toggling** — a set-UID program can:
1. Start with effective UID = file owner (e.g., root).
2. Drop to real UID for normal operations (`seteuid(getuid())`).
3. Re-elevate to saved set-UID when privileged operations are needed (`seteuid(0)`).
4. Permanently drop privilege by setting all three to the real UID (`setuid(getuid())` as root).

### Set-UID and Set-GID Programs

When a binary has the set-UID bit (`chmod u+s`), executing it sets the process's effective UID to the file's owner rather than the invoking user. Common set-UID-root programs: `passwd`, `su`, `sudo`, `ping`.

**Security best practice**: minimize the window of elevated privilege. Drop privileges immediately after performing the operation that requires them. Never run user-supplied input or call `exec()` while holding elevated privileges.

## File Permissions

The standard Unix permission model:

```
-rwxr-x--- 1 alice developers 4096 Apr 09 10:00 script.sh
 │││ │││ │││
 │││ │││ └── other: no access
 │││ └──┘── group: read + execute
 └──┘────── owner: read + write + execute
```

- **Read (r)** — files: read content; directories: list entries.
- **Write (w)** — files: modify content; directories: create/delete entries.
- **Execute (x)** — files: run as program; directories: traverse (access entries by name).

### The Sticky Bit

When set on a directory (`chmod +t`), the sticky bit prevents users from deleting or renaming files they don't own, even if they have write permission to the directory. This is how `/tmp` works — anyone can create files, but only the owner (or root) can delete them.

### umask

The `umask` controls the default permissions for newly created files and directories. It is a bitmask of permission bits to **remove** from the default. For example, `umask 022` removes group-write and other-write, so new files get `644` and directories `755`.

## POSIX Capabilities

Capabilities decompose the all-or-nothing root privilege into fine-grained units. A process can hold specific capabilities without being root:

| Capability | Permits |
|------------|---------|
| `CAP_NET_BIND_SERVICE` | Bind to ports below 1024 |
| `CAP_NET_RAW` | Use raw sockets (e.g., `ping`) |
| `CAP_SYS_ADMIN` | Broad system admin operations (mount, sethostname, etc.) |
| `CAP_DAC_OVERRIDE` | Bypass file read/write/execute permission checks |
| `CAP_DAC_READ_SEARCH` | Bypass read and directory search permissions |
| `CAP_CHOWN` | Change file ownership arbitrarily |
| `CAP_KILL` | Send signals to any process |
| `CAP_SETUID` / `CAP_SETGID` | Change UID/GID arbitrarily |
| `CAP_SYS_PTRACE` | Trace any process with ptrace |
| `CAP_SYS_CHROOT` | Use `chroot()` |
| `CAP_IPC_LOCK` | Lock memory pages (`mlock`, `mlockall`) |
| `CAP_SYS_NICE` | Raise scheduling priority, set real-time policy |

Capabilities can be set on executables (file capabilities) via `setcap`, eliminating the need for set-UID root in many cases. For example: `setcap cap_net_bind_service=ep /usr/bin/myserver`.

## chroot Jails

`chroot(path)` changes the process's view of the filesystem root to `path`. After `chroot("/srv/jail")`, the process sees `/srv/jail` as `/` and cannot access anything outside it.

**Limitations**: `chroot` is not a complete security sandbox:
- A root process inside a chroot can escape via `mknod`, `mount`, creating a new chroot, or using file descriptors opened before the chroot.
- It only restricts filesystem access — it does not limit network access, IPC, signals, or system calls.
- Modern alternatives for stronger isolation: namespaces (mount, PID, network), seccomp-BPF, containers.

Proper chroot usage: drop root privileges after chrooting, close all file descriptors to the outside, `chdir("/")` after chrooting.

## PAM (Pluggable Authentication Modules)

PAM provides a framework for abstracting authentication from applications:

- **Configuration**: `/etc/pam.d/<service>` files define module stacks with control flags (`required`, `sufficient`, `optional`, `requisite`).
- **Module types**: `auth` (verify identity), `account` (check access), `password` (update credentials), `session` (setup/teardown).
- **Conversation function**: the mechanism by which PAM modules interact with the user (prompt for password, display messages). Applications implement a conversation function callback.
- **Common modules**: `pam_unix` (traditional `/etc/shadow` passwords), `pam_ldap`, `pam_deny`/`pam_permit`, `pam_limits`, `pam_env`.

PAM allows changing authentication methods (add two-factor, switch to LDAP) without modifying applications.

## Security Best Practices

Principles from "Advanced Linux Programming" for writing secure applications:

1. **Principle of least privilege** — request only the capabilities you need, drop them as soon as possible.
2. **Validate all input** — never trust data from users, files, network, or environment variables. Especially dangerous: `PATH`, `LD_PRELOAD`, `IFS`.
3. **Avoid TOCTOU races** — don't check access then use; instead, attempt the operation and handle failure. Use `O_EXCL` for exclusive creation.
4. **Avoid `system()` and `popen()`** — they invoke a shell, enabling injection attacks. Use `fork()`+`exec()` with explicit argument arrays.
5. **Clean the environment** — sanitize or clear environment variables before exec'ing other programs.
6. **Use `mkstemp()`** — for temporary files, never `tempnam()` or `mktemp()` (which have TOCTOU race conditions).
7. **Check return values** — every system call and library function can fail; ignoring errors creates security vulnerabilities.

## See also

- [process-management](../entities/process-management.md)
- [virtual-filesystem](../entities/virtual-filesystem.md)
- [proc-filesystem](../entities/proc-filesystem.md)
- [system-calls](../entities/system-calls.md)
