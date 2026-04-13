---
type: entity
created: 2026-04-08
updated: 2026-04-08
sources: [understanding-the-linux-kernel]
tags: [device-model, sysfs, kobject, device-drivers, pci, dma]
---

# Linux Device Driver Model

The Linux 2.6 device driver model provides a unified framework for representing devices, drivers, and buses in the kernel. It replaces the ad-hoc per-subsystem approaches of Linux 2.4 with a coherent hierarchy that supports power management, hot-plugging, and user-space visibility via [sysfs](virtual-filesystem.md). The model is built on a small set of core abstractions -- kobjects, ksets, and subsystems -- that export the device hierarchy to user space through the sysfs virtual filesystem.

## Core Abstractions

### kobject

The **`struct kobject`** is the fundamental building block of the device model. Every object that needs reference counting, sysfs representation, or hierarchical naming embeds a kobject.

Key fields:

- `k_name` -- pointer to the object's name (shown as a directory name in sysfs)
- `kref` -- reference count (`struct kref`), using `kobject_get()` and `kobject_put()` to increment and decrement. When the count reaches zero, the object's `release()` function is called.
- `entry` -- linkage into the kset's list
- `parent` -- pointer to the parent kobject, establishing the hierarchy
- `kset` -- pointer to the containing kset
- `ktype` -- pointer to `struct kobj_type`, which provides:
  - `release()` -- destructor called when refcount hits zero
  - `sysfs_ops` -- `show()` and `store()` functions for reading/writing sysfs attributes
  - `default_attrs` -- array of default attributes (files) created in the sysfs directory
- `dentry` -- pointer to the sysfs directory entry for this kobject

Every kobject corresponds to a directory in sysfs. When a kobject is registered via `kobject_register()`, a directory is created under its parent's sysfs directory.

### kset

A **`struct kset`** is a collection of kobjects of the same type. It forms a sysfs directory containing its member kobjects as subdirectories.

Key fields:

- `kobj` -- embedded kobject (the kset itself is a kobject, giving it a sysfs directory)
- `list` -- list of member kobjects
- `ktype` -- default kobj_type for members
- `hotplug_ops` -- hotplug/uevent operations for notifying user-space (via udev) when objects are added or removed:
  - `filter()` -- decide whether to generate a hotplug event
  - `name()` -- provide the subsystem name for the event
  - `hotplug()` -- add environment variables to the event

A kset acts as a container and event dispatcher: adding a kobject to a kset triggers hotplug notifications to user space.

### Subsystem

A **subsystem** is a top-level grouping that adds mutual exclusion to a kset:

- `kset` -- the embedded kset
- `rwsem` -- read-write semaphore protecting the subsystem's data

Subsystems represent the major categories visible at the top level of sysfs: `bus/`, `devices/`, `class/`, `block/`, etc. Each is registered via `subsystem_register()`.

## Device Model Abstractions

### bus_type

A **`struct bus_type`** represents a bus (PCI, USB, platform, I2C, etc.):

- `name` -- bus name (e.g., "pci", "usb")
- `match()` -- function that checks if a given driver supports a given device. The bus subsystem calls this during driver binding.
- `hotplug()` -- add bus-specific environment variables for hotplug events
- `probe()` -- called when a match is found (optional; usually the driver's probe is used)
- `remove()` -- called when a device is unbound from its driver
- `suspend()` / `resume()` -- power management callbacks
- `devices` -- kset of all devices on this bus
- `drivers` -- kset of all drivers registered for this bus

When a new device is discovered or a new driver is loaded, the bus iterates over its devices/drivers and calls `match()` to find compatible pairs. Successful matches trigger the driver's `probe()` function.

### device

A **`struct device`** represents a device in the system:

- `bus_id` -- unique identifier on the bus
- `bus` -- pointer to the bus_type
- `driver` -- pointer to the bound driver (NULL if unbound)
- `parent` -- parent device (e.g., a PCI device's parent is the PCI bus, whose parent is the bridge)
- `kobj` -- embedded kobject
- `power` -- power management state
- `release()` -- destructor
- `driver_data` -- opaque pointer for driver-private data
- `dma_mask` -- DMA addressing capability

Devices form a tree rooted at a virtual "system" device. This tree reflects the physical bus topology and is visible under `/sys/devices/`.

### device_driver

A **`struct device_driver`** represents a driver:

- `name` -- driver name
- `bus` -- the bus type this driver works with
- `probe()` -- called when a matching device is found. The driver initializes the device, allocates resources, and registers interfaces. Returns 0 on success, negative on failure.
- `remove()` -- called when the device is removed or the driver is unloaded. Must release all resources.
- `shutdown()` -- called during system shutdown for orderly device cleanup.
- `suspend()` / `resume()` -- power management callbacks
- `kobj` -- embedded kobject
- `devices` -- list of devices bound to this driver

The lifecycle: `driver_register()` adds the driver to its bus's driver kset. The bus's `match()` function is called for each unbound device. For each match, `probe()` is called. If `probe()` succeeds, the device-driver binding is established.

## sysfs

**sysfs** is a virtual filesystem (mounted at `/sys`) that exports the device model hierarchy to user space. It is built entirely on kobjects: each kobject is a directory, and each attribute is a file.

### Directory Structure

```
/sys/
├── bus/          # one subdir per bus type
│   ├── pci/
│   │   ├── devices/    # symlinks to /sys/devices/...
│   │   └── drivers/    # one subdir per PCI driver
│   └── usb/
├── devices/      # physical device tree
│   └── pci0000:00/
│       └── 0000:00:1f.0/
├── class/        # devices grouped by function (net, input, sound)
│   ├── net/
│   │   └── eth0 -> ../../devices/.../net/eth0
│   └── input/
├── block/        # block devices
│   ├── sda/
│   └── sdb/
├── module/       # loaded kernel modules
└── firmware/     # firmware objects
```

### Attributes

Kobject attributes appear as regular files in sysfs. Reading a file invokes the `show()` method; writing invokes the `store()` method. Attributes are defined as `struct attribute` (name + mode) and registered via `sysfs_create_file()`. Device drivers commonly define custom attributes to expose device-specific parameters.

Binary attributes (`struct bin_attribute`) support arbitrary-length data, used for things like PCI configuration space or firmware blobs.

## PCI Subsystem

PCI is the most important bus in typical Linux systems. The PCI subsystem registers with the device model and provides PCI-specific matching and resource management.

### pci_dev

**`struct pci_dev`** represents a PCI device:

- `vendor`, `device` -- PCI vendor and device IDs
- `class` -- PCI class code
- `subsystem_vendor`, `subsystem_device` -- subsystem IDs
- `irq` -- assigned IRQ number
- `resource[DEVICE_COUNT_RESOURCE]` -- array of I/O and memory resources (BARs)
- `dev` -- embedded `struct device`
- `bus` -- pointer to `struct pci_bus`
- `devfn` -- encoded device and function number

### pci_driver

**`struct pci_driver`** represents a PCI driver:

- `name` -- driver name
- `id_table` -- array of `struct pci_device_id` for matching
- `probe()` -- called when a matching device is found
- `remove()` -- called on device removal
- `suspend()` / `resume()` -- power management
- `driver` -- embedded `struct device_driver`

### PCI Device ID Matching

**`struct pci_device_id`** specifies which devices a driver supports:

- `vendor`, `device` -- exact match or `PCI_ANY_ID` for wildcard
- `subvendor`, `subdevice` -- subsystem IDs or `PCI_ANY_ID`
- `class`, `class_mask` -- class matching with mask
- `driver_data` -- opaque driver-private value

The PCI bus's `match()` function iterates over the driver's `id_table` entries, checking each against the device's IDs. The `MODULE_DEVICE_TABLE(pci, ...)` macro exports the ID table for use by user-space module loading (via modprobe and `modules.pcimap`).

### PCI Configuration Space

Every PCI device has a 256-byte (or 4096-byte for PCI Express) configuration space accessible via `pci_read_config_byte/word/dword()` and `pci_write_config_byte/word/dword()`. This space contains standard registers (vendor/device ID, BARs, interrupt line, status, command) and device-specific registers.

## I/O Access Methods

### I/O Port Access

x86 processors have a separate 64K I/O port address space accessed via special `in` and `out` instructions. The kernel provides:

- `inb(port)`, `inw(port)`, `inl(port)` -- read 8/16/32 bits from a port
- `outb(value, port)`, `outw(value, port)`, `outl(value, port)` -- write to a port
- `insb()`, `insw()`, `insl()` -- string (repeated) input
- `outsb()`, `outsw()`, `outsl()` -- string output

### I/O Port Reservation

Before using I/O ports, a driver must reserve them to prevent conflicts:

- **`request_region(start, n, name)`** -- reserves `n` ports starting at `start`. Returns a `struct resource *` on success, NULL if already reserved. Reservations are visible in `/proc/ioports`.
- **`release_region(start, n)`** -- releases the reservation.
- **`check_region(start, n)`** -- deprecated; checks availability without reserving.

### Memory-Mapped I/O (MMIO)

Many devices map their registers into the physical address space. The kernel provides:

- **`ioremap(phys_addr, size)`** -- maps a physical address range into kernel virtual address space. Returns a `void __iomem *`. The mapping is uncacheable to ensure that every read/write reaches the device.
- **`iounmap(addr)`** -- releases the mapping.
- `readb(addr)`, `readw(addr)`, `readl(addr)` -- read 8/16/32 bits from MMIO
- `writeb(value, addr)`, `writew(value, addr)`, `writel(value, addr)` -- write to MMIO

MMIO reservations use `request_mem_region()` and `release_mem_region()`, visible in `/proc/iomem`.

## Resource Management

The kernel maintains a **resource tree** to track I/O port and memory allocations, preventing device conflicts. The tree uses `struct resource`:

- `start`, `end` -- address range
- `name` -- descriptive name
- `flags` -- `IORESOURCE_IO`, `IORESOURCE_MEM`, `IORESOURCE_IRQ`, etc.
- `parent`, `child`, `sibling` -- tree linkage

The tree is hierarchical: a PCI bus occupies a range, and devices on that bus occupy sub-ranges within it. The functions `request_resource()`, `allocate_resource()`, and `release_resource()` manage the tree. `request_region()` and `request_mem_region()` are convenience wrappers around `request_resource()`.

## Device Files

User-space programs access devices through **device files** (device nodes) in `/dev`. Each device file is identified by:

- **Type**: character (stream-oriented: terminals, mice, sound) or block (random-access: disks, partitions)
- **Major number**: identifies the driver or device class (e.g., 8 = SCSI disks)
- **Minor number**: identifies a specific device or partition within the class (e.g., minor 0 = sda, minor 1 = sda1)

Device numbers are encoded in `dev_t` using `MKDEV(major, minor)` and decoded with `MAJOR(dev)` and `MINOR(dev)`. In Linux 2.6, device numbers are 32 bits: 12-bit major (0--4095) and 20-bit minor (0--1048575).

### Character Device Registration

Character devices are registered using the `cdev` interface:

1. Allocate a `struct cdev` via `cdev_alloc()` or embed it in the driver's private structure.
2. Initialize it with `cdev_init(cdev, &fops)`, providing the `file_operations` dispatch table.
3. Register it with `cdev_add(cdev, dev, count)`, specifying the first device number and count.

Alternatively, the older `register_chrdev(major, name, &fops)` registers a driver for an entire major number (all 256 minors). This is simpler but wastes minor number space.

When user space opens a character device file, the kernel:

1. Looks up the `cdev` by major/minor in the `cdev_map` hash table.
2. Retrieves the `file_operations` from the `cdev`.
3. Sets `file->f_op` to the driver's operations.
4. Calls `file->f_op->open()`.

All subsequent `read()`, `write()`, `ioctl()`, and `mmap()` calls dispatch through the driver's `file_operations`, just as for regular files in the [virtual-filesystem](virtual-filesystem.md).

## DMA (Direct Memory Access)

DMA allows devices to transfer data directly to/from memory without CPU involvement, essential for high-bandwidth I/O.

### Address Types

Three address spaces are involved:

- **CPU physical addresses** -- what the CPU uses to address RAM
- **Bus addresses** -- what the device uses to address RAM via the bus (may differ from physical addresses)
- **Virtual addresses** -- what the kernel uses (mapped from physical via page tables)

On systems without an IOMMU (I/O Memory Management Unit), bus addresses equal physical addresses. With an IOMMU, bus addresses are translated through a separate page table, allowing:

- Devices with limited address ranges (32-bit) to access all physical memory
- Scatter-gather I/O where physically non-contiguous pages appear contiguous to the device
- I/O protection and isolation

### Coherent DMA Mappings

**`dma_alloc_coherent(dev, size, &dma_handle, gfp)`** allocates a buffer accessible by both CPU and device without explicit cache synchronization. Returns a kernel virtual address and sets `dma_handle` to the bus address. Used for long-lived shared structures (descriptor rings, command queues). Freed with `dma_free_coherent()`.

### Streaming DMA Mappings

**`dma_map_single(dev, vaddr, size, direction)`** maps an existing kernel buffer for one-time DMA. `direction` is `DMA_TO_DEVICE`, `DMA_FROM_DEVICE`, or `DMA_BIDIRECTIONAL`. The CPU must not access the buffer while mapped. Call `dma_unmap_single()` after transfer. Scatter-gather variants: `dma_map_sg()` / `dma_unmap_sg()`.

### Bounce Buffers and DMA Masks

If a device cannot address all physical memory (e.g., 32-bit PCI with >4 GB RAM), the kernel allocates **bounce buffers** in DMA-reachable memory. The device's DMA capability is declared via `dma_set_mask(dev, mask)`. The [block-layer](block-layer.md) uses bounce buffers transparently when a bio's pages are above the device's DMA mask.

## See also

- [virtual-filesystem](virtual-filesystem.md)
- [block-layer](block-layer.md)
- [ext2-ext3](ext2-ext3.md)
- [page-cache](page-cache.md)
