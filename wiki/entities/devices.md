---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [advanced-linux-programming, understanding-the-linux-kernel]
tags: [devices, character-devices, block-devices, dev, ioctl, hardware]
---

# Devices

Linux follows the Unix principle of "everything is a file" — hardware devices are represented as special files (device nodes) in the `/dev` directory. Applications interact with hardware by opening these files and using standard file operations (`read`, `write`, `ioctl`), while the kernel routes operations to the appropriate device driver.

## Device Types

| Type | Access pattern | Examples | Kernel representation |
|------|---------------|----------|----------------------|
| **Character** | Sequential byte stream, no seeking (usually) | Terminals, serial ports, `/dev/null`, `/dev/random`, sound cards | `cdev`, registered via `register_chrdev()` |
| **Block** | Random-access fixed-size blocks, buffered through page cache | Hard disks, SSDs, partitions, loopback devices | `gendisk`, managed by the [block-layer](block-layer.md) |

Each device is identified by a **major number** (identifies the driver) and a **minor number** (identifies the specific device instance managed by that driver). The `ls -l /dev` output shows these numbers where the file size would normally appear.

## Device Nodes

Device nodes are special filesystem entries created with `mknod`:

```
mknod /dev/mydevice c 42 0    # character device, major 42, minor 0
mknod /dev/mydisk   b 8  1    # block device, major 8, minor 1
```

On modern Linux systems, device nodes are typically managed automatically by **udev** (or **devtmpfs**), which creates and removes nodes in response to hardware events.

## Special Devices

Several character devices provide kernel services rather than representing physical hardware:

| Device | Purpose |
|--------|---------|
| `/dev/null` | Discards all data written to it. Reads always return EOF. Used to suppress output: `cmd > /dev/null 2>&1` |
| `/dev/zero` | Reads return an infinite stream of zero bytes. Writes are discarded. Used with `mmap()` for zero-initialized anonymous memory, or with `dd` to create files filled with zeros |
| `/dev/full` | Reads return zero bytes (like `/dev/zero`). Writes always fail with `ENOSPC` ("No space left on device"). Useful for testing error handling |
| `/dev/random` | Cryptographically secure random bytes from the kernel entropy pool. Blocks when entropy is exhausted (pre-5.6 kernels) |
| `/dev/urandom` | Like `/dev/random` but never blocks — reuses the internal entropy pool. Suitable for almost all applications; `/dev/random` blocking is rarely needed |
| `/dev/loop*` | Loopback devices — present a regular file as a block device. Used for mounting filesystem images: `mount -o loop image.iso /mnt` |

## Pseudo-Terminals (PTYs)

PTYs provide a pair of connected character devices — a **master** side and a **slave** side — that create a virtual terminal. Data written to the master appears as input on the slave, and vice versa. The slave side behaves like a real terminal, supporting terminal I/O processing (line editing, signal generation via Ctrl+C, etc.).

- **Allocation**: opening `/dev/ptmx` (the PTY multiplexer) allocates a new PTY pair and returns a file descriptor for the master side. `ptsname()` returns the slave path (e.g., `/dev/pts/3`).
- **Use cases**: terminal emulators (xterm, gnome-terminal), SSH sessions, `screen`/`tmux`, `expect`.
- **Line discipline**: the slave side passes through the kernel's terminal line discipline, which provides canonical mode (line buffering, backspace handling) and non-canonical mode (raw input).

## The ioctl Interface

The `ioctl()` system call provides a catch-all mechanism for device-specific operations that don't map to `read`/`write`/`seek`:

```c
int ioctl(int fd, unsigned long request, ...);
```

Each driver defines its own set of request codes and argument types. Examples:

| Device | ioctl | Purpose |
|--------|-------|---------|
| Terminal | `TIOCGWINSZ` | Get terminal window size (rows, columns) |
| Terminal | `TIOCSWINSZ` | Set terminal window size |
| CD-ROM | `CDROMEJECT` | Eject disc |
| Block device | `BLKGETSIZE64` | Get device size in bytes |
| Network | `SIOCGIFADDR` | Get interface IP address |

The `ioctl` interface is powerful but type-unsafe — the kernel has no way to validate that the argument matches what the driver expects. Modern kernel development prefers alternatives like sysfs attributes, netlink sockets, or dedicated file-based interfaces where possible.

## Relationship to Kernel Internals

The kernel side of device management is covered in [device-driver-model](device-driver-model.md), which describes the `kobject`/sysfs hierarchy, bus/device/driver abstractions, and DMA. The [block-layer](block-layer.md) covers request queues and I/O schedulers for block devices. The [proc-filesystem](proc-filesystem.md) and sysfs (`/sys`) provide alternative views of device information.

## See also

- [device-driver-model](device-driver-model.md)
- [block-layer](block-layer.md)
- [proc-filesystem](proc-filesystem.md)
- [virtual-filesystem](virtual-filesystem.md)
