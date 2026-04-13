---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [advanced-linux-programming]
tags: [proc, virtual-filesystem, kernel-interface, debugging, monitoring]
---

# The /proc File System

The `/proc` file system is a **virtual filesystem** — it contains no on-disk data. Instead, its files and directories are generated on-the-fly by the kernel to expose runtime information about processes, hardware, and kernel state. Reading from `/proc` entries invokes kernel functions that format and return the current state; writing to certain entries modifies kernel parameters at runtime.

Mounted at `/proc` (mount type `proc`), it is the kernel's primary user-space introspection interface. Tools like `ps`, `top`, `free`, `lsmod`, and `sysctl` are thin wrappers around `/proc` reads.

## Per-Process Entries: /proc/\<pid\>/

Each running process has a directory `/proc/<pid>/` containing virtual files that describe the process's state. The special symlink `/proc/self` always points to the current process's directory.

### Key Process Files

| File | Content |
|------|---------|
| `status` | Human-readable process status: name, state (Running/Sleeping/Zombie/etc.), TGID, PID, PPid, UIDs (real/effective/saved/fs), GIDs, VmSize/VmRSS/VmData/VmStk, signal masks (SigPnd, SigBlk, SigIgn, SigCgt), capability sets, voluntary/involuntary context switches |
| `cmdline` | The command line used to launch the process. Arguments are null-separated (`\0`). Empty for zombie processes and kernel threads |
| `environ` | Environment variables, null-separated (`KEY=value\0KEY2=value2\0`) |
| `exe` | Symlink to the executable binary. Resolves to `(deleted)` suffix if the binary has been removed |
| `cwd` | Symlink to the process's current working directory |
| `root` | Symlink to the process's root directory (different from `/` if `chroot()` was used) |
| `maps` | Memory mappings (VMAs): address range, permissions (`rwxp`/`rwxs`), offset, device, inode, and pathname for each region. Shows code, data, heap, stack, shared libraries, and anonymous mappings |
| `fd/` | Directory of symlinks, one per open file descriptor. `fd/0` → stdin, `fd/1` → stdout, etc. The link target reveals the actual file, socket, or pipe |
| `stat` | Raw process statistics in a single space-separated line: PID, comm, state, ppid, pgrp, session, tty, minflt, majflt, utime, stime, priority, nice, num_threads, starttime, vsize, rss, and more |
| `statm` | Memory statistics in pages: total, resident, shared, text, data+stack |

### Reading Process Information

Process entries are the foundation of system monitoring. For example:

- **`ps aux`** reads `stat`, `status`, `cmdline` for each `/proc/<pid>/`
- **`top`** repeatedly reads `stat` and `statm` for CPU and memory usage
- **`lsof`** reads `fd/` to find open files per process
- **`/proc/self/maps`** lets a program inspect its own memory layout

## Hardware Information

| Path | Content |
|------|---------|
| `/proc/cpuinfo` | Per-CPU details: model name, vendor, frequency, cache size, feature flags (sse, sse2, mmx, etc.), physical/core IDs, bogomips |
| `/proc/interrupts` | IRQ assignment table: interrupt number, per-CPU counts, controller type (IO-APIC, PIC), device name |
| `/proc/ioports` | I/O port address ranges claimed by drivers |
| `/proc/iomem` | Physical memory address ranges for devices and RAM |
| `/proc/dma` | DMA channel assignments |
| `/proc/pci` or `/proc/bus/pci/` | PCI device information (largely superseded by `/sys/bus/pci/`) |

## Kernel and System Information

| Path | Content |
|------|---------|
| `/proc/version` | Kernel version string, compiler version, build date |
| `/proc/modules` | Loaded kernel modules: name, size, use count, dependencies |
| `/proc/filesystems` | Filesystem types known to the kernel (marked `nodev` for virtual filesystems) |
| `/proc/mounts` | Currently mounted filesystems (equivalent to `/etc/mtab` but always accurate) |
| `/proc/uptime` | Two numbers: seconds since boot, seconds spent idle (summed across CPUs) |
| `/proc/loadavg` | 1-minute, 5-minute, 15-minute load averages, then running/total processes and last PID assigned |
| `/proc/meminfo` | Detailed memory breakdown: MemTotal, MemFree, MemAvailable, Buffers, Cached, SwapTotal, SwapFree, Active, Inactive, Dirty, Writeback, and many more |
| `/proc/vmstat` | Virtual memory statistics: page faults, pageins/outs, swap activity, allocation events |
| `/proc/stat` | System-wide statistics: per-CPU time breakdown (user, nice, system, idle, iowait, irq, softirq), context switches, boot time, process forks, procs_running, procs_blocked |

## Tunable Kernel Parameters: /proc/sys/

The `/proc/sys/` subtree contains writable files that control kernel behavior at runtime. These are the same parameters accessed by the `sysctl` command.

| Subtree | Controls |
|---------|----------|
| `/proc/sys/kernel/` | Core kernel parameters: `hostname`, `osrelease`, `pid_max`, `threads-max`, `shmmax`, `msgmax`, `panic`, `randomize_va_space` |
| `/proc/sys/vm/` | Virtual memory tuning: `swappiness`, `dirty_ratio`, `dirty_background_ratio`, `overcommit_memory`, `min_free_kbytes`, `vfs_cache_pressure` |
| `/proc/sys/net/` | Networking parameters: `ip_forward`, TCP buffer sizes, connection timeouts, congestion control |
| `/proc/sys/fs/` | Filesystem limits: `file-max` (system-wide FD limit), `inode-max`, `pipe-max-size` |

Changes via `/proc/sys/` take effect immediately but do not persist across reboots. For persistence, use `/etc/sysctl.conf` or files in `/etc/sysctl.d/`.

## Relationship to Kernel Internals

The `/proc` filesystem is implemented as part of the [virtual-filesystem](virtual-filesystem.md) layer — it registers as a filesystem type with specific `read`/`write` operations that call into kernel subsystems to generate content. Process-specific entries are created and destroyed as processes start and exit, managed by [process-management](process-management.md).

The `/proc/sys/` interface is backed by the `sysctl` mechanism in the kernel, with each tunable registered as a `ctl_table` entry.

## See also

- [virtual-filesystem](virtual-filesystem.md)
- [process-management](process-management.md)
- [memory-management](memory-management.md)
- [device-driver-model](device-driver-model.md)
- [kernel-modules](kernel-modules.md)
