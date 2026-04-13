---
type: entity
created: 2026-04-08
updated: 2026-04-08
sources: [linux-kernel-in-a-nutshell]
tags: [kconfig, menuconfig, configuration, config, kernel-options]
---

# Kernel Configuration (Kconfig)

Kconfig is the kernel's configuration infrastructure. It defines a hierarchical set of configuration options (over 3,000 in a typical 2.6 kernel), manages dependencies between them, and produces a `.config` file that drives the [Kbuild](kernel-build-system.md) build system. Every `CONFIG_` variable in the kernel originates from a Kconfig definition.

## Configuration Files

### .config

The `.config` file in the kernel source root is the master configuration. Each line takes one of three forms:

```
CONFIG_SMP=y                # Built into the kernel image
CONFIG_EXT3_FS=m            # Built as a loadable module
# CONFIG_REISERFS_FS is not set  # Disabled
```

This file is consumed by the build system to determine which source files to compile and how (built-in vs. module).

### Kconfig Source Files

Each directory in the kernel source contains a `Kconfig` file that defines the options relevant to that subsystem. The syntax supports:

- **config**: Defines a single option with type (bool, tristate, int, string, hex), prompt text, default value, dependencies, and help text
- **menuconfig**: Like config but creates a collapsible submenu
- **menu/endmenu**: Groups options under a titled heading
- **choice/endchoice**: Mutually exclusive option groups
- **depends on**: Gate visibility on another option
- **select**: Force-enable another option when this one is enabled
- **source**: Include another Kconfig file

### Tristate Options

Many kernel options are **tristate** -- they have three possible values:

| Value | Meaning | In .config |
|-------|---------|------------|
| **Y** | Built directly into the kernel image (vmlinux) | `CONFIG_X=y` |
| **M** | Built as a loadable [module](kernel-modules.md) (.ko file) | `CONFIG_X=m` |
| **N** | Not built at all | `# CONFIG_X is not set` |

Boolean options only support Y or N. The choice between Y and M involves a trade-off: built-in code is always available and slightly faster (no module loading overhead), but increases kernel image size and memory footprint. Modules are loaded on demand and can be unloaded, but require module infrastructure.

## Configuration Tools

### make menuconfig (ncurses TUI)

The most widely used configuration tool. Presents a hierarchical menu system in the terminal:

- Navigate with arrow keys, Enter to descend into submenus
- Press **Y** to set an option to built-in
- Press **M** to set to module
- Press **N** to disable
- Press **?** for help text describing the option
- Press **/** to search for an option by name

Searching is particularly useful: entering `EXT3` in the search dialog shows where `CONFIG_EXT3_FS` lives in the menu hierarchy, its current value, and its dependencies.

### make xconfig (Qt GUI)

Graphical configuration using the Qt toolkit. Provides a split-pane view with the menu tree on the left and option details on the right. Requires Qt development libraries installed.

### make gconfig (GTK+ GUI)

Similar to xconfig but uses GTK+ instead of Qt. Choose based on which toolkit is available on the build system.

### make config (Line-by-Line)

Asks every configuration question sequentially. No backtracking -- if you answer incorrectly, you must start over. Rarely used except in scripts.

### make oldconfig

Reuses an existing `.config` and prompts only for **new** options not present in the current config. This is the standard workflow for kernel upgrades:

```sh
# Copy old config to new kernel source
cp /boot/config-2.6.18 /usr/src/linux-2.6.19/.config
cd /usr/src/linux-2.6.19
make oldconfig    # Answer only new questions
```

### make silentoldconfig

Like `oldconfig` but suppresses output for unchanged options, showing only truly new questions. Useful in automated build scripts.

### make defconfig

Generates a default configuration for the target architecture. The defaults are maintained by architecture maintainers in `arch/<ARCH>/defconfig` or `arch/<ARCH>/configs/`. Produces a working kernel for reference hardware but requires customization for specific systems.

### make allmodconfig / allyesconfig / allnoconfig

| Target | Description |
|--------|-------------|
| `allmodconfig` | Set every tristate option to M (module); useful for compile-testing |
| `allyesconfig` | Set everything to Y (built-in); maximum kernel |
| `allnoconfig` | Set everything to N; minimal kernel |

## Practical Configuration Workflows

### Starting from a Distribution Kernel

The fastest path to a working custom kernel:

1. Obtain the running kernel's config:
   ```sh
   # Method 1: /proc/config.gz (if CONFIG_IKCONFIG_PROC=y)
   zcat /proc/config.gz > .config
   # Method 2: Copy from /boot
   cp /boot/config-$(uname -r) .config
   ```
2. Run `make oldconfig` to adapt to the new kernel version
3. Use `make menuconfig` to customize

### Starting from Scratch

1. Run `make defconfig` for a baseline
2. Identify hardware using sysfs, `lspci`, and `lsusb`
3. Enable drivers for detected hardware in `menuconfig`
4. Enable needed filesystems, networking features, and security options

### Device Discovery Workflow

The book describes a systematic approach to finding needed kernel modules:

1. **List PCI devices**: `lspci` shows all PCI devices with vendor/device IDs
2. **Find sysfs entries**: `/sys/bus/pci/devices/<BDF>/` contains device attributes
3. **Identify the driver**: check if a `driver` symlink exists under the sysfs device directory
4. **Find the module**: if `driver` exists, `readlink` shows the driver name; search kernel Makefiles for the corresponding `CONFIG_` option
5. **Enable in menuconfig**: search with `/` for the option name

For USB devices, the same workflow uses `lsusb` and `/sys/bus/usb/devices/`.

A helper script can automate the process by iterating through sysfs device directories, following driver symlinks, and extracting module names.

## Major Configuration Categories

### General Setup

- `CONFIG_LOCALVERSION` -- Append a custom string to the kernel version
- `CONFIG_SWAP` -- Enable swap space support
- `CONFIG_SYSVIPC` -- System V IPC (shared memory, semaphores, message queues; see [ipc](ipc.md))
- `CONFIG_SYSCTL` -- `/proc/sys` interface for runtime kernel parameter tuning
- `CONFIG_IKCONFIG` / `CONFIG_IKCONFIG_PROC` -- Embed `.config` in the kernel, accessible via `/proc/config.gz`
- `CONFIG_MODULES` -- Enable [loadable module](kernel-modules.md) support
- `CONFIG_MODULE_UNLOAD` -- Allow module unloading
- `CONFIG_MODVERSIONS` -- Symbol versioning for module ABI compatibility

### Processor Configuration

- `CONFIG_SMP` -- Symmetric multiprocessing support
- `CONFIG_NR_CPUS` -- Maximum number of CPUs supported
- `CONFIG_PREEMPT_NONE` -- No kernel preemption (server workloads, max throughput)
- `CONFIG_PREEMPT_VOLUNTARY` -- Voluntary preemption points (desktop, lower latency)
- `CONFIG_PREEMPT` -- Full kernel preemption (low-latency desktop/embedded)
- `CONFIG_HIGHMEM4G` / `CONFIG_HIGHMEM64G` -- High memory support for >1 GB / >4 GB RAM on 32-bit
- `CONFIG_NOHIGHMEM` -- No high memory (machines with <=1 GB RAM)

### Power Management

- `CONFIG_PM` -- Power management framework (APM and/or ACPI)
- `CONFIG_ACPI` -- ACPI support (replaces APM, PnP BIOS, MPS)
- `CONFIG_SOFTWARE_SUSPEND` -- Suspend-to-disk (hibernation)
- `CONFIG_CPU_FREQ` -- CPU frequency scaling with multiple governors:
  - **performance** -- always maximum frequency
  - **powersave** -- always minimum frequency
  - **ondemand** -- dynamic scaling based on CPU utilization
  - **conservative** -- gradual frequency changes (battery-friendly)
  - **userspace** -- manual control from user space

### Filesystems

- `CONFIG_EXT2_FS` / `CONFIG_EXT3_FS` -- Second/third extended filesystem (see [ext2-ext3](ext2-ext3.md))
- `CONFIG_REISERFS_FS` -- ReiserFS (balanced tree, good for small files)
- `CONFIG_XFS_FS` -- XFS (high-performance, extent-based, from SGI)
- `CONFIG_JFS_FS` -- JFS (IBM's journaled filesystem)
- `CONFIG_FUSE_FS` -- FUSE (filesystem in userspace)
- `CONFIG_INOTIFY` -- File change notification (replacement for dnotify)
- `CONFIG_NFS_FS` -- NFS client support
- `CONFIG_SMB_FS` / `CONFIG_CIFS` -- Windows file sharing (SMB/CIFS)

### Kernel Hacking / Debugging

- `CONFIG_DEBUG_KERNEL` -- Master switch for debugging options
- `CONFIG_DEBUG_FS` -- debugfs virtual filesystem for developer debugging files
- `CONFIG_MAGIC_SYSRQ` -- SysRq key for emergency system control
- `CONFIG_PRINTK_TIME` -- Timestamps on printk messages (useful for boot profiling)
- `CONFIG_KPROBES` -- Dynamic kernel probing for instrumentation
- `CONFIG_OPROFILE` -- System-wide profiling
- `CONFIG_PROFILING` -- Extended profiling support

### Security

- `CONFIG_SECURITY` -- Enable Linux Security Module (LSM) framework
- `CONFIG_SECURITY_SELINUX` -- NSA Security-Enhanced Linux (mandatory access control)
- `CONFIG_SECCOMP` -- Secure computing mode (restrict syscalls for sandboxing)

## See also

- [kernel-build-system](kernel-build-system.md)
- [kernel-modules](kernel-modules.md)
- [boot-process](boot-process.md)
- [device-driver-model](device-driver-model.md)
- [src-linux-kernel-in-a-nutshell](../sources/src-linux-kernel-in-a-nutshell.md)
