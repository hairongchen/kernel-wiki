---
type: entity
created: 2026-04-08
updated: 2026-04-08
sources: [understanding-the-linux-kernel, linux-kernel-in-a-nutshell]
tags: [boot, startup, bios, grub, initialization, boot-parameters]
---

# Linux Boot Process

The Linux boot process is a multi-stage sequence that transforms a powered-off machine into a running kernel with user-space processes. Each stage operates under different constraints — real mode, protected mode without paging, protected mode with paging — and hands off control to the next stage after establishing progressively richer execution environments.

## Stage 1: BIOS

When power is applied to an x86 system, the processor begins executing at physical address 0xFFFFFFF0, which is mapped to a ROM containing the BIOS. The BIOS performs several critical tasks:

### Power-On Self-Test (POST)

The BIOS runs POST to verify the integrity of essential hardware: the CPU, memory controller, RAM, and basic I/O devices. Failures at this stage produce diagnostic beep codes since video hardware may not yet be initialized.

### Hardware Inventory and ACPI Tables

The BIOS probes the system bus to enumerate installed devices and constructs **ACPI tables** (Advanced Configuration and Power Interface) that describe the hardware topology, interrupt routing, power management capabilities, and memory maps. These tables are placed in reserved memory regions where the kernel will later read them. Key ACPI tables include the RSDP (Root System Description Pointer), RSDT/XSDT, MADT (for interrupt controllers), and FADT (for power management).

### Boot Device Selection

The BIOS consults its configured boot order (stored in CMOS NVRAM) to determine which device to boot from — typically a hard disk, floppy, CD-ROM, or network interface. For a disk boot, the BIOS reads the first 512-byte sector (the Master Boot Record, or MBR) from the selected device and loads it into physical memory at address **0x00007C00**. The BIOS then jumps to that address to transfer control to the boot loader.

At this point the processor is still in **real mode** — a 16-bit execution environment with 1 MB of addressable memory, no memory protection, and segmented addressing.

## Stage 2: Boot Loader (LILO/GRUB)

The boot loader's job is to load the Linux kernel image from disk into memory and transfer control to it. Linux 2.6 systems typically use one of two boot loaders:

### LILO (Linux Loader)

LILO is a simpler, older boot loader that operates in two stages. The MBR code (stage 1) loads a second-stage loader that knows how to read the kernel image from specific disk sectors recorded at LILO installation time. LILO does not understand filesystems — it relies on a precomputed sector map.

### GRUB (Grand Unified Bootloader)

GRUB is more sophisticated. Its stage 1 code in the MBR loads a stage 1.5 (filesystem-aware driver) from the sectors immediately following the MBR, which in turn loads the full stage 2 from a filesystem. GRUB understands ext2/3, FAT, and other filesystems, allowing it to locate the kernel image by pathname rather than raw sector numbers. GRUB also provides an interactive menu and command line.

### Loading the Kernel Image

The boot loader loads the kernel image into memory at one of two addresses depending on the image type:

| Image Type | Load Address | Maximum Size | Description |
|------------|-------------|--------------|-------------|
| `zImage` | **0x00010000** (low memory, 64 KB) | 512 KB | Legacy "small" compressed kernel |
| `bzImage` | **0x00100000** (high memory, 1 MB) | No practical limit | "Big" compressed kernel (standard) |

The `bzImage` format is used for modern kernels because the compressed kernel plus decompression code easily exceeds the 512 KB limit of `zImage`. The "b" in `bzImage` stands for "big."

The boot loader also loads the kernel's **setup code** (the first few sectors of the kernel image) into memory starting at 0x00090200. The boot parameters (command line, memory information from the BIOS) are placed at 0x00090000.

## Stage 3: setup() — Real Mode to Protected Mode

The `setup()` function is an assembly routine located at offset **0x200** (512 bytes) into the kernel image, in `arch/i386/boot/setup.S`. It serves as the bridge between the 16-bit real-mode BIOS world and the 32-bit protected-mode kernel.

### Hardware Reinitialization

The kernel does not trust the BIOS to have left hardware in a known state. `setup()` reinitializes key hardware components:

- **Video adapter**: Queries and optionally sets the display mode.
- **Disk controller**: Resets the disk subsystem to a known state.

### Memory Detection via e820

`setup()` invokes BIOS interrupt **INT 0x15, function 0xE820** to obtain a detailed physical memory map. This is the "e820" memory map, which reports each region of physical address space along with its type:

- **Usable RAM** — available for kernel use
- **Reserved** — used by BIOS or firmware
- **ACPI Reclaimable** — holds ACPI tables; can be reused after the kernel parses them
- **ACPI NVS** — non-volatile storage; must not be touched

This map is essential because physical memory on x86 is not contiguous — there are holes for legacy devices (640 KB-1 MB), memory-mapped I/O, and ACPI regions.

### Enabling the A20 Line

On the original IBM PC, address line A20 (bit 20 of the physical address) was forced to zero, limiting addressing to 1 MB and providing wrap-around compatibility with 8086 software. `setup()` enables the **A20 line** (also called the A20 gate) so that the full 32-bit address space is accessible. This is done through the keyboard controller (port 0x64/0x60), the "fast A20" port (0x92), or a BIOS call, depending on the platform.

### Programming the PICs

`setup()` reprograms the two **8259A Programmable Interrupt Controllers** (PICs). In real mode, the BIOS maps the 8 hardware IRQs from the master PIC to interrupt vectors 0x08-0x0F, which collide with x86 exception vectors (divide error, debug, NMI, breakpoint, etc.). The kernel remaps:

- **Master PIC** (IRQ 0-7) to vectors **0x20-0x27**
- **Slave PIC** (IRQ 8-15) to vectors **0x28-0x2F**

This avoids conflict between hardware interrupts and CPU exceptions.

### Provisional GDT and IDT

`setup()` creates a minimal **Global Descriptor Table** (GDT) with basic code and data segments and a null **Interrupt Descriptor Table** (IDT). These are provisional — the kernel will replace them with final versions later. The provisional GDT contains flat segments (base 0, limit 4 GB) so that logical addresses equal physical addresses, which simplifies the transition.

### Switching to Protected Mode

The climactic step: `setup()` sets the **PE (Protection Enable) bit** in control register **CR0**, switching the CPU from real mode to protected mode. This enables:

- 32-bit addressing (4 GB address space)
- Segment-based memory protection via the GDT
- Privilege levels (rings 0-3)

A far jump immediately follows the mode switch to flush the prefetch queue and load the CS register with a protected-mode selector. Control passes to the first `startup_32()` function.

## Stage 4: First startup_32() — Kernel Decompression

The first `startup_32()` resides in `arch/i386/boot/compressed/head.S`. This code runs in protected mode but operates on the still-compressed kernel image.

### Decompression

This function sets up a minimal environment (stack, segments) and calls **`decompress_kernel()`**, which implements gzip or bzip2 decompression. This is the stage that prints the familiar message:

```
Uncompressing Linux... Ok, booting the kernel.
```

The decompressed kernel image is placed at the appropriate physical address (typically 0x00100000 for a bzImage). If the compressed and decompressed images would overlap, the code first copies the compressed image to a safe location before decompressing.

After decompression, execution jumps to the second `startup_32()` in the decompressed kernel.

## Stage 5: Second startup_32() — Kernel Initialization in Protected Mode

The second `startup_32()` lives in `arch/i386/kernel/head.S` and is the true entry point of the decompressed kernel. It establishes the execution environment that C code requires.

### Final Segmentation

This function loads the final GDT and segment registers with proper kernel-mode selectors. The provisional GDT from `setup()` is discarded. The final GDT includes segments for kernel code, kernel data, user code, user data, TSS (Task State Segment), and LDT (Local Descriptor Table) entries.

### Zeroing the BSS

The BSS (Block Started by Symbol) segment — which holds uninitialized global and static variables — is zeroed out. C code assumes BSS variables start as zero, so this step is mandatory.

### Provisional Page Tables

`startup_32()` constructs provisional page tables anchored at **`swapper_pg_dir`** (the kernel's page global directory). These identity-map the first portion of physical memory (enough to cover the kernel image) so that physical and virtual addresses coincide during the transition to paging. The kernel also maps the same physical memory at the kernel's virtual address range (starting at 0xC0000000, i.e., PAGE_OFFSET).

### Enabling Paging

The function sets the **PG (Paging) bit** in **CR0**, which activates the [memory-management](memory-management.md) unit's address translation. After this point, all memory references go through page table translation. The register CR3 is loaded with the physical address of `swapper_pg_dir`.

### Kernel Stack for Process 0

`startup_32()` sets up the **kernel-mode stack** for process 0 (the idle/swapper process). The stack pointer (ESP) is loaded with the address of `init_thread_union + 8192` — pointing to the top of a two-page (8 KB) stack that is statically allocated. This thread_union structure holds both the `thread_info` structure and the kernel stack for process 0.

### Jump to C Code

Finally, `startup_32()` calls **`start_kernel()`**, the first C function in the boot sequence.

## Stage 6: start_kernel() — Subsystem Initialization

`start_kernel()` in `init/main.c` orchestrates the initialization of virtually every kernel subsystem. It is a long sequence of function calls, each setting up one piece of the kernel. Key calls include (in approximate order):

### Core Infrastructure

- **`setup_arch()`** — Architecture-specific initialization: parses the e820 memory map, sets up the kernel's physical memory map, initializes the fixmap area, and configures NUMA topology if applicable.
- **`setup_per_cpu_areas()`** — Allocates per-CPU data areas for SMP systems.
- **`build_all_zonelists()`** — Constructs the ordered zone lists used by the [buddy system](memory-management.md) page allocator. Zones include ZONE_DMA (0-16 MB), ZONE_NORMAL (16 MB-896 MB), and ZONE_HIGHMEM (above 896 MB).

### Traps and Interrupts

- **`trap_init()`** — Sets up the final **IDT** with handlers for CPU exceptions: divide error (vector 0), debug (1), NMI (2), breakpoint (3), overflow (4), page fault (14), general protection fault (13), and others.
- **`init_IRQ()`** — Initializes the interrupt handling framework and populates the remaining IDT entries for hardware [interrupt-handling](interrupt-handling.md).

### Memory Management

- **`mem_init()`** — Finalizes the memory management subsystem: frees the boot memory allocator's pages to the buddy allocator, reports total available memory, and marks reserved pages.
- **`kmem_cache_init()`** — Initializes the [slab allocator](memory-management.md) (SLAB cache), which provides efficient allocation of small, frequently-used kernel objects. After this call, `kmalloc()` becomes functional.
- **`vfs_caches_init()`** — Sets up dcache, inode cache, and other [virtual-filesystem](virtual-filesystem.md) caches.

### Scheduler and Timing

- **`sched_init()`** — Initializes the [process-scheduler](process-scheduler.md) run queues and creates the data structures for the O(1) scheduler. Process 0's scheduling parameters are set.
- **`time_init()`** — Calibrates the system timer and sets up the [timekeeping](timing-subsystem.md) infrastructure.
- **`calibrate_delay()`** — Measures the **BogoMIPS** value — a rough indication of processor speed used for short busy-wait delay loops (`udelay()`). The result is printed during boot (e.g., "Calibrating delay loop... 4800.00 BogoMIPS").

### Console and Filesystems

- **`console_init()`** — Activates the kernel console so that `printk()` output becomes visible.
- **`rest_init()`** — The final function called by `start_kernel()`, which creates process 1 and enters the idle loop.

## Stage 7: Process 1 and User Space

### Creating Process 1

`rest_init()` calls **`kernel_thread()`** to spawn **process 1**, which executes the `init()` function (later renamed `kernel_init()`). Process 0 then becomes the **idle process**, executing the idle loop forever — it runs only when no other process is runnable.

### Process 1: init()

Process 1 (still running in kernel mode) performs final setup:

1. Calls `do_basic_setup()`, which triggers all compiled-in `initcall` functions (device drivers, subsystem initializers registered via `module_init()`).
2. Frees the `__init` memory — code and data marked with `__init` are only needed during boot and are discarded to reclaim memory.
3. Opens `/dev/console` as file descriptors 0, 1, and 2 (stdin, stdout, stderr).
4. Attempts to execute a user-space init program, trying in order:
   - The path specified by the `init=` boot parameter
   - `/sbin/init`
   - `/etc/init`
   - `/bin/init`
   - `/bin/sh` (last resort)

If none succeeds, the kernel panics with "No init found."

### initrd and initramfs

For systems that need special drivers to mount the root filesystem (SCSI controllers, LVM, encrypted root, etc.), the boot loader can load an **initial RAM disk (initrd)** or **initial RAM filesystem (initramfs)** alongside the kernel image.

- **initrd**: A compressed filesystem image loaded into a ramdisk. The kernel mounts it as a temporary root filesystem, executes `/linuxrc` within it to load necessary [kernel-modules](kernel-modules.md), and then performs `pivot_root()` to switch to the real root filesystem.
- **initramfs** (preferred in 2.6): A cpio archive unpacked directly into a [virtual-filesystem](virtual-filesystem.md) (tmpfs-based rootfs). It is simpler and more flexible than initrd — the kernel unpacks the archive into rootfs and runs `/init` from it. The initramfs approach avoids the double-caching problem of initrd (where data exists in both the ramdisk and the page cache).

## Summary of Address Space Transitions

| Stage | Mode | Addressing | Key Event |
|-------|------|-----------|-----------|
| BIOS | Real mode (16-bit) | Segmented, 1 MB max | POST, hardware init |
| Boot loader | Real mode | Segmented, 1 MB max | Kernel loaded to memory |
| setup() | Real -> Protected | PE bit set in CR0 | A20 enabled, GDT loaded |
| First startup_32() | Protected, no paging | Flat 4 GB | Kernel decompressed |
| Second startup_32() | Protected + paging | Virtual addresses | PG bit set in CR0 |
| start_kernel() | Protected + paging | Full virtual memory | All subsystems initialized |

## Kernel Boot Parameters

The kernel accepts command-line parameters from the boot loader that configure hardware, override defaults, and control early-boot behavior. Parameters are passed as `key=value` pairs or bare keywords in the boot loader configuration (e.g., GRUB's `kernel` line or `linux` line).

### Core Parameters

| Parameter | Description |
|-----------|-------------|
| `root=` | Root filesystem device (e.g., `root=/dev/sda1`, `root=LABEL=rootfs`, `root=UUID=...`) |
| `ro` / `rw` | Mount root filesystem read-only (default, for fsck) or read-write |
| `init=` | Path to the init program (default: `/sbin/init`). Use `init=/bin/sh` for emergency recovery. |
| `rootfstype=` | Filesystem type of root device (e.g., `ext3`, `xfs`) |
| `rootdelay=` | Seconds to wait before mounting root (for slow USB/SCSI devices) |
| `quiet` | Suppress most boot messages (only errors and warnings) |
| `debug` | Enable verbose kernel debug messages |
| `panic=N` | Reboot N seconds after a kernel panic (0 = never reboot) |
| `mem=N` | Override detected physical memory size (e.g., `mem=512M`, `mem=2G`) |

### Console Parameters

| Parameter | Description |
|-----------|-------------|
| `console=` | Output device for kernel messages (e.g., `console=ttyS0,115200` for serial, `console=tty0` for VGA). Multiple consoles can be specified; the last one becomes `/dev/console`. |
| `earlyprintk=` | Enable early console before the normal console driver initializes |
| `loglevel=N` | Set the default printk log level (0-7; lower = more critical only) |
| `log_buf_len=N` | Set the kernel log buffer size |

### Module and Init Parameters

| Parameter | Description |
|-----------|-------------|
| `initrd=` | Path to initial ramdisk image |
| `noinitrd` | Do not use any initrd even if the boot loader loaded one |
| `load_ramdisk=` | Load ramdisk image from floppy |
| `module.param=value` | Pass parameter to a built-in module (e.g., `usbcore.autosuspend=-1`) |

### CPU and Memory Parameters

| Parameter | Description |
|-----------|-------------|
| `nosmp` | Disable SMP; boot as a uniprocessor system |
| `max_cpus=N` | Limit the number of active CPUs |
| `isolcpus=` | Isolate CPUs from the scheduler (for real-time or dedicated use) |
| `highmem=` / `nohighmem` | Control high memory usage on 32-bit systems |
| `noexec=on` | Enable NX (no-execute) bit for memory protection |
| `noapic` / `nolapic` | Disable I/O APIC or local APIC |
| `lpj=N` | Override loops-per-jiffy calibration (skip `calibrate_delay()`) |

### Other Parameters

| Parameter | Description |
|-----------|-------------|
| `isolcpus=` | CPU list to isolate from general SMP balancing |
| `clocksource=` | Select timekeeping clocksource (e.g., `tsc`, `hpet`, `acpi_pm`) |
| `initcall_debug` | Print timing for each initcall during boot |
| `crashkernel=` | Reserve memory for kdump crash kernel (used with kexec) |
| `nmi_watchdog=` | Configure NMI watchdog for detecting hard lockups |

Full reference: `Documentation/kernel-parameters.txt` in the kernel source.

## See also

- [memory-management](memory-management.md)
- [interrupt-handling](interrupt-handling.md)
- [process-scheduler](process-scheduler.md)
- [slab allocator](memory-management.md)
- [buddy system](memory-management.md)
- [kernel-modules](kernel-modules.md)
- [virtual-filesystem](virtual-filesystem.md)
- [kernel-configuration](kernel-configuration.md)
- [kernel-build-system](kernel-build-system.md)
- [segmentation](../concepts/concept-memory-addressing.md)
- [paging](../concepts/concept-memory-addressing.md)
- [timekeeping](timing-subsystem.md)
