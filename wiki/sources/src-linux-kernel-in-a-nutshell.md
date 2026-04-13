---
type: source
created: 2026-04-08
updated: 2026-04-08
sources: [linux-kernel-in-a-nutshell]
tags: [source, kernel-build, kconfig, kbuild, configuration, installation]
---

# Linux Kernel in a Nutshell

**Author:** Greg Kroah-Hartman
**Publisher:** O'Reilly Media, 2007
**Scope:** Practical guide to building, configuring, installing, and upgrading the Linux kernel (2.6.x series)

Greg Kroah-Hartman is the maintainer of the USB, PCI, driver core, and sysfs subsystems in the kernel, and a member of the -stable kernel release team. This book is a hands-on companion to the more theoretical "Understanding the Linux Kernel" -- where Bovet & Cesati explain how the kernel works internally, Kroah-Hartman explains how to build, configure, and install it.

## Part I: Building the Kernel (Chapters 1--7)

### Chapter 1: Introduction

Overview of the Linux kernel's development model. The kernel uses a time-based release cycle with:
- **Stable releases** (e.g., 2.6.19) on the main line
- **-stable patches** (e.g., 2.6.19.1) for critical bug fixes backported from development
- **-rc releases** (release candidates) during the merge window

### Chapter 2: Requirements

Tools needed to build the kernel:
- **gcc** (C compiler) -- version must match kernel requirements
- **binutils** (as, ld) -- the assembler and linker
- **make** (3.79.1+) -- build orchestration
- **util-linux**, **module-init-tools** (modprobe, insmod, depmod), **e2fsprogs**, and filesystem-specific tools (jfsutils, reiserfsprogs, xfsprogs, quota-tools, nfs-utils, udev, procfs)

### Chapter 3: Retrieving the Kernel Source

Sources available from kernel.org. Kernel source trees are identified by `VERSION.PATCHLEVEL.SUBLEVEL.EXTRAVERSION` in the top-level Makefile. Patches are applied with the `patch` utility; `ketchup` automates downloading and applying patches.

### Chapter 4: Configuring the Kernel (Kconfig)

The kernel uses the **Kconfig** infrastructure for configuration. Key tools:

| Tool | Interface | Description |
|------|-----------|-------------|
| `make config` | Line-by-line | Asks every option one at a time (no backtracking) |
| `make menuconfig` | ncurses TUI | Console-based, hierarchical menus. Most popular. |
| `make xconfig` | Qt GUI | Graphical, requires Qt libraries |
| `make gconfig` | GTK+ GUI | Graphical, requires GTK+ libraries |
| `make oldconfig` | Text | Reuses existing `.config`, prompts only for new options |
| `make silentoldconfig` | Text | Like oldconfig but only shows new questions |
| `make defconfig` | Automatic | Creates a default configuration for the architecture |

Configuration state is stored in the `.config` file at the kernel source root. Each option is `CONFIG_<NAME>=y` (built-in), `CONFIG_<NAME>=m` (module), or `# CONFIG_<NAME> is not set`.

The book provides practical techniques for customizing a kernel:
- **Starting from a distribution kernel**: extract running kernel's config from `/proc/config.gz` or `/boot/config-<version>`, then run `make oldconfig`
- **Starting from scratch**: use `make defconfig` then customize
- **Finding needed modules**: use `lsmod`, `lspci`, `lsusb`, and sysfs to discover hardware, then search Makefiles for `CONFIG_` rules that build the corresponding modules

### Chapter 5: Building the Kernel (Kbuild)

The kernel uses the **Kbuild** build system. Key make targets:

| Target | Purpose |
|--------|---------|
| `make` / `make all` | Build vmlinux, modules, and bzImage |
| `make -jN` | Parallel build using N jobs (speeds build on SMP) |
| `make O=/path` | Out-of-tree build (source and output in different directories) |
| `make ARCH=<arch>` | Cross-compile for a different architecture |
| `make CROSS_COMPILE=<prefix>` | Cross-compiler prefix (e.g., `arm-linux-gnueabi-`) |
| `make M=/path` | Build only modules in the specified directory |
| `make dir/` | Build files only in that subdirectory |

Advanced build options:
- **ccache**: Compiler cache for faster rebuilds
- **distcc**: Distributed compilation across machines
- `make EXTRAVERSION=-custom`: Append string to kernel version

### Chapter 6: Installing the Kernel

Installation steps:
1. `make modules_install` -- installs modules to `/lib/modules/<version>/`
2. Copy `arch/x86/boot/bzImage` to `/boot/vmlinuz-<version>`
3. Copy `System.map` to `/boot/`
4. Modify bootloader config (GRUB: `/boot/grub/menu.lst`; LILO: `/etc/lilo.conf`)
5. If needed, generate `initrd`/`initramfs` image via `mkinitrd` or `mkinitramfs`

The `make install` target automates steps 2-5 on many distributions by running the distribution's install scripts.

### Chapter 7: Upgrading a Kernel

Techniques for upgrading:
- Download new source or apply incremental patches (`patch -p1 < patch-2.6.20`)
- Run `make oldconfig` to update `.config` for the new version
- Use `git bisect` to find the exact commit that introduced a regression
- Automate with `ketchup` for downloading and patching kernel sources

## Part II: Major Customization (Chapters 8--11)

### Chapter 8: Disk Device Configuration

Practical guide to configuring disk controller support:
- **IDE/ATA disks**: Determining controller with `lspci`, enabling `CONFIG_IDE`, chipset-specific drivers
- **SATA (Serial ATA)**: Uses the libata library, configure via `CONFIG_SCSI` + `CONFIG_SCSI_SATA` + chipset driver
- **USB storage**: `CONFIG_USB` + `CONFIG_USB_STORAGE` + HCD driver (EHCI/OHCI/UHCI)
- **CD-ROM drives**: IDE (`CONFIG_BLK_DEV_IDECD`) or SCSI (`CONFIG_BLK_DEV_SR`)

### Chapter 9: USB Device Configuration

Finding USB devices via `lsusb` and sysfs. Practical device discovery: using `/sys/bus/usb/devices/` to match vendor/product IDs to drivers. Covers wireless USB device driver discovery.

### Chapter 10: Network Device Configuration

Network device discovery via `lspci`, `ifconfig`, and sysfs. Main Kconfig options: `CONFIG_NET`, `CONFIG_INET`, `CONFIG_NETFILTER`, `CONFIG_NETDEVICES`, `CONFIG_NET_ETHERNET`. Covers wireless (`CONFIG_NET_RADIO`, `CONFIG_IEEE80211`).

### Chapter 11: Kernel Configuration Option Reference

Comprehensive reference of major Kconfig options organized by category:

- **General setup**: `CONFIG_LOCALVERSION`, `CONFIG_SWAP`, `CONFIG_SYSVIPC`, `CONFIG_BSD_PROCESS_ACCT`, `CONFIG_SYSCTL`, `CONFIG_IKCONFIG`, `CONFIG_MODULES`
- **Loadable module support**: `CONFIG_MODULE_UNLOAD`, `CONFIG_MODULE_FORCE_UNLOAD`, `CONFIG_MODVERSIONS`
- **Processor types and features**: `CONFIG_SMP`, `CONFIG_PREEMPT_NONE/_VOLUNTARY/_BKL/PREEMPT`, `CONFIG_NOHIGHMEM`/`CONFIG_HIGHMEM4G`/`CONFIG_HIGHMEM64G`
- **Power management**: `CONFIG_PM`, `CONFIG_ACPI`, `CONFIG_SOFTWARE_SUSPEND`, `CONFIG_CPU_FREQ` (governors: performance, powersave, userspace, ondemand, conservative)
- **Bus options**: `CONFIG_PCI`, `CONFIG_PCCARD`/`CONFIG_PCMCIA`/`CONFIG_CARDBUS`, `CONFIG_HOTPLUG_PCI`
- **Networking**: `CONFIG_NET`, `CONFIG_UNIX`, `CONFIG_INET`, `CONFIG_IP_ADVANCED_ROUTER`, `CONFIG_NETFILTER`, `CONFIG_NET_SCHED`, `CONFIG_IRDA`, `CONFIG_BT`, `CONFIG_IEEE80211`
- **Device drivers**: `CONFIG_IDE`, `CONFIG_SCSI`, `CONFIG_SCSI_SATA`, `CONFIG_MD`, `CONFIG_BLK_DEV_DM`, `CONFIG_IEEE1394`, `CONFIG_NETDEVICES`, `CONFIG_INPUT`, `CONFIG_VT`, `CONFIG_SERIAL_8250`, `CONFIG_AGP`, `CONFIG_DRM`, `CONFIG_I2C`, `CONFIG_SPI`, `CONFIG_HWMON`, `CONFIG_VIDEO_DEV`, `CONFIG_FB`, `CONFIG_SOUND`/`CONFIG_SND`, `CONFIG_USB`, `CONFIG_USB_GADGET`, `CONFIG_MMC`, `CONFIG_INFINIBAND`, `CONFIG_EDAC`
- **Filesystems**: `CONFIG_EXT2_FS`, `CONFIG_EXT3_FS`, `CONFIG_REISERFS_FS`, `CONFIG_JFS_FS`, `CONFIG_XFS_FS`, `CONFIG_OCFS2_FS`, `CONFIG_INOTIFY`, `CONFIG_QUOTA`, `CONFIG_AUTOFS_FS`, `CONFIG_FUSE_FS`, `CONFIG_SMB_FS`, `CONFIG_CIFS`
- **Kernel hacking**: `CONFIG_DEBUG_KERNEL`, `CONFIG_DEBUG_FS`, `CONFIG_PROFILING`, `CONFIG_OPROFILE`, `CONFIG_KPROBES`, `CONFIG_PRINTK_TIME`, `CONFIG_MAGIC_SYSRQ`
- **Security**: `CONFIG_SECURITY`, `CONFIG_SECURITY_SELINUX`

## Part IV: Additional Information

### Appendix A: Helpful Utilities

- **patch/diff**: Standard tools for creating and applying kernel patches; use `-Naur` flags for recursive diffs
- **quilt**: Patch management tool for maintaining series of patches against a source tree. Supports push/pop/refresh workflow.
- **git**: Distributed version control used by the Linux kernel project. `git bisect` is particularly valuable for finding regressions.
- **ketchup**: Automated tool for downloading, verifying (GPG signatures), and applying kernel patches to switch between versions

### Appendix B: Bibliography

References to kernel programming books (Linux Device Drivers, Linux Kernel Development, Understanding the Linux Kernel) and tool download locations.

## Key Themes

1. **Practical over theoretical**: The book focuses on "how to do it" rather than "how it works internally." Complements Bovet & Cesati's theoretical treatment.
2. **Device discovery workflow**: Systematic approach using `lspci`/`lsusb`/sysfs to identify hardware, find corresponding kernel modules, and enable correct `CONFIG_` options.
3. **Configuration as the key skill**: The book argues that most kernel work is configuration work -- choosing the right options for the target hardware.
4. **Kconfig/Kbuild as infrastructure**: Detailed treatment of the build system that other kernel books largely skip.
5. **Distribution integration**: Practical advice on integrating custom kernels with distribution boot infrastructure (GRUB/LILO, initrd, module installation).

## See also

- [kernel-build-system](../entities/kernel-build-system.md)
- [kernel-configuration](../entities/kernel-configuration.md)
- [kernel-modules](../entities/kernel-modules.md)
- [boot-process](../entities/boot-process.md)
- [device-driver-model](../entities/device-driver-model.md)
- [src-understanding-the-linux-kernel](src-understanding-the-linux-kernel.md)
