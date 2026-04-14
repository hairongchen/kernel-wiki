---
type: analysis
created: 2026-04-09
updated: 2026-04-09
sources: [qemu-kvm-source-code-and-application, mastering-kvm-virtualization]
tags: [kvm, vfio, iommu, pci-passthrough, vt-d, amd-vi, sr-iov, device-assignment, dma-remapping]
---

# VFIO and IOMMU Device Passthrough

VFIO (Virtual Function I/O) is the Linux kernel framework that enables safe, direct assignment of physical devices to userspace programs — most importantly, to QEMU/KVM virtual machines. It relies on the IOMMU (I/O Memory Management Unit) to provide DMA isolation and interrupt remapping, ensuring that a passed-through device can only access memory explicitly mapped for it.

Device passthrough eliminates the hypervisor from the I/O data path. Instead of QEMU emulating a virtual device or even using paravirtual virtio, the guest VM drives the real hardware directly with its native driver. This delivers near-native I/O performance — critical for GPUs, high-speed NICs, NVMe storage, and FPGA accelerators.

## IOMMU Hardware

### Intel VT-d and AMD-Vi

Both Intel and AMD provide IOMMU implementations:

| Feature | Intel VT-d | AMD-Vi (AMD IOMMU) |
|---------|-----------|-------------------|
| DMA remapping | Multi-level page tables | Multi-level page tables |
| Interrupt remapping | Interrupt Remapping Table (IRT) | Interrupt Remapping Table |
| I/O page faults | Posted Page Request (PPR) — VT-d 3.0 | PPR (AMD IOMMUv2) |
| Specification | VT-d spec (Intel) | AMD I/O Virtualization spec |

Both provide the same fundamental capabilities; Linux abstracts them behind a common IOMMU API (`drivers/iommu/`).

### DMA Remapping

Without an IOMMU, a PCIe device performing DMA can read or write any physical address on the system. This is catastrophic for isolation: a device assigned to one VM could DMA into another VM's memory or into the hypervisor.

The IOMMU interposes a **second-level page table** between the device and physical memory, analogous to how EPT/NPT translates guest-physical to host-physical addresses for the CPU:

```
Device DMA request
  │
  ▼
┌─────────────────┐
│  IOMMU           │  Translates device bus address (IOVA)
│  Page Tables     │  to host physical address (HPA)
│                  │  using per-domain page tables
└─────────────────┘
  │
  ▼
Host Physical Memory
```

Each IOMMU **domain** has its own set of page tables. A device assigned to a KVM guest gets a domain whose page tables map the guest's RAM — the device sees IOVAs (I/O Virtual Addresses) that correspond to guest-physical addresses, and the IOMMU translates them to host-physical addresses. The device cannot access any memory outside this mapping.

### Interrupt Remapping

Without interrupt remapping, a malicious or buggy device could forge MSI/MSI-X interrupt messages to inject interrupts into arbitrary CPUs or vectors — effectively a DMA attack via the interrupt path. Interrupt remapping forces all device-generated interrupts through an **Interrupt Remapping Table** (IRT) maintained by the hypervisor:

1. The device generates an MSI/MSI-X write to the interrupt address range.
2. The IOMMU intercepts the write and uses the message's handle field to index into the IRT.
3. The IRT entry specifies the actual destination CPU, vector, and delivery mode.
4. The IOMMU reformats and delivers the interrupt according to the IRT entry.

This prevents a device from targeting interrupts at CPUs or vectors it is not authorized to use.

### Boot-Time Configuration

IOMMU support must be enabled via kernel boot parameters:

```
# Intel systems
intel_iommu=on

# AMD systems
amd_iommu=on

# Passthrough mode: use IOMMU only for devices explicitly assigned to VMs,
# not for host device DMA (avoids performance overhead on host drivers)
iommu=pt
```

The `iommu=pt` option is recommended for KVM hosts. It configures the IOMMU in passthrough mode for host devices (no translation overhead) while still enabling full IOMMU isolation for devices assigned via VFIO.

Verify IOMMU is active:

```bash
dmesg | grep -i iommu
# Intel: "DMAR: IOMMU enabled"
# AMD:   "AMD-Vi: AMD IOMMUv2 functionality not available on this system" or similar
```

## IOMMU Groups

### The Isolation Boundary

An IOMMU group is the smallest set of devices that the IOMMU can isolate from all other devices. It is the fundamental unit of device assignment — you cannot assign a single device from a group; the entire group must be assigned together.

IOMMU groups exist because PCIe topology can allow peer-to-peer transactions between devices that bypass the IOMMU:

```
                ┌─────────┐
                │  Root    │
                │ Complex  │
                └────┬────┘
                     │
              ┌──────┴──────┐
              │   PCIe      │
              │   Switch    │     ← If switch lacks ACS, devices
              ├──────┬──────┤       behind it can peer-to-peer DMA
              │      │      │       and MUST be in the same group
           ┌──┴──┐┌──┴──┐┌──┴──┐
           │Dev A││Dev B││Dev C│
           └─────┘└─────┘└─────┘
              IOMMU Group N
```

### Access Control Services (ACS)

PCIe ACS is a capability that prevents peer-to-peer transactions between devices behind the same switch or bridge without going through the root complex (where the IOMMU sits). If a PCIe switch supports ACS, the kernel can place each device behind it in its own IOMMU group. Without ACS, all devices behind the switch are grouped together.

### Discovering IOMMU Groups

```bash
# List all IOMMU groups
ls /sys/kernel/iommu_groups/

# Show devices in a specific group
ls /sys/kernel/iommu_groups/14/devices/
# 0000:03:00.0  0000:03:00.1

# Find which group a device belongs to
readlink /sys/bus/pci/devices/0000:03:00.0/iommu_group
# ../../../kernel/iommu_groups/14
```

A multi-function device (e.g., a NIC with separate PCI functions for ports) typically has all functions in the same group. Assigning one function requires assigning all of them.

## VFIO Kernel Framework

VFIO provides a secure, userspace-accessible interface to devices via three layers of abstraction:

```
┌─────────────────────────────────────┐
│  Userspace (QEMU)                   │
│                                     │
│  Container fd (/dev/vfio/vfio)      │  ← IOMMU context: DMA mappings
│    └── Group fd (/dev/vfio/<N>)     │  ← IOMMU group: all devices in group
│          └── Device fd              │  ← Individual device: BARs, interrupts
└─────────────────────────────────────┘
```

### Container

A VFIO container (`/dev/vfio/vfio`) represents an IOMMU context — a set of DMA page table mappings. Opening the container device and setting the IOMMU type (typically `VFIO_TYPE1_IOMMU`) establishes the translation context. The container is where DMA mappings are programmed:

- `VFIO_IOMMU_MAP_DMA` — map an IOVA range to a userspace virtual address (the kernel pins the pages and programs the IOMMU page tables)
- `VFIO_IOMMU_UNMAP_DMA` — remove a mapping

### Group

Each IOMMU group is represented by a device node `/dev/vfio/<group_id>`. Opening the group fd and attaching it to a container links all devices in that group to the container's IOMMU context. All devices in a group must be either: (a) bound to a VFIO driver (e.g., `vfio-pci`), or (b) bound to a known-safe stub driver, or (c) unbound — before the group can be used.

### Device

Individual devices are obtained from the group via `VFIO_GROUP_GET_DEVICE_FD`. The device fd exposes:

| ioctl | Purpose |
|-------|---------|
| `VFIO_DEVICE_GET_INFO` | Number of regions (BARs) and interrupt types |
| `VFIO_DEVICE_GET_REGION_INFO` | Size, offset, and flags for each BAR |
| `VFIO_DEVICE_GET_IRQ_INFO` | Interrupt types (INTx, MSI, MSI-X) and count |
| `VFIO_DEVICE_SET_IRQS` | Configure interrupt delivery (eventfd-based) |
| `VFIO_DEVICE_RESET` | Reset the device |

Device BARs can be accessed via `read()`/`write()` on the device fd (slow, trapped), or via `mmap()` at the offset returned by `VFIO_DEVICE_GET_REGION_INFO` (fast, direct MMIO access from userspace).

### Binding a Device to vfio-pci

Before VFIO can manage a device, the device must be unbound from its host driver and bound to the `vfio-pci` driver:

```bash
# Load the VFIO modules
modprobe vfio-pci

# Identify the device
lspci -nn -s 0000:03:00.0
# 03:00.0 Ethernet controller [0200]: Intel Corporation 82599ES [8086:10fb]

# Unbind from host driver
echo 0000:03:00.0 > /sys/bus/pci/devices/0000:03:00.0/driver/unbind

# Bind to vfio-pci (using vendor:device ID)
echo "8086 10fb" > /sys/bus/pci/drivers/vfio-pci/new_id

# Verify
ls -la /dev/vfio/
# 14  vfio    (group 14 appeared)
```

After binding, the device is no longer available to the host and can be assigned to a VM.

## QEMU/KVM Integration

When QEMU is launched with `-device vfio-pci,host=0000:03:00.0`, the following sequence occurs:

### 1. Device Acquisition

```
QEMU opens /dev/vfio/vfio                    → container fd
QEMU sets VFIO_TYPE1_IOMMU on container
QEMU opens /dev/vfio/<group_id>              → group fd
QEMU attaches group to container              (VFIO_GROUP_SET_CONTAINER)
QEMU gets device fd from group                (VFIO_GROUP_GET_DEVICE_FD)
```

### 2. DMA Mapping (Guest RAM)

QEMU maps the guest's entire RAM into the IOMMU via `VFIO_IOMMU_MAP_DMA`. Each KVM memory slot is mapped so that the device's DMA addresses correspond to guest-physical addresses:

```
VFIO_IOMMU_MAP_DMA:
  iova  = guest_physical_address    (what the device will DMA to)
  vaddr = qemu_virtual_address      (QEMU's mmap of guest RAM)
  size  = memory_region_size
```

The kernel pins the guest RAM pages and programs the IOMMU page tables. The device can now DMA directly to guest memory using guest-physical addresses — exactly as the guest's native driver expects.

### 3. BAR Mapping

QEMU reads the device's BAR information via `VFIO_DEVICE_GET_REGION_INFO` and maps the BARs into the guest's physical address space as MMIO regions. When the guest driver accesses a device register:

- **MMIO BARs mapped via mmap**: Guest accesses go through EPT to the MMIO region that QEMU has mmap'd from the VFIO device fd. The access reaches the physical device directly — no VM Exit required for data-plane register accesses.
- **Trapped regions**: Some BAR regions (e.g., PCI config space) are trapped by KVM, causing a VM Exit so QEMU can emulate or mediate the access.

### 4. Interrupt Delivery

VFIO configures device interrupts using eventfds that integrate with KVM's irqfd mechanism:

```
Physical device raises MSI-X interrupt
  → VFIO receives the interrupt (via the host's interrupt handler)
  → VFIO signals the registered eventfd
  → KVM's irqfd sees the eventfd signal
  → KVM injects the interrupt into the guest
  → (With APICv/posted interrupts: delivered without VM Exit)
```

This is the same irqfd mechanism used by [vhost](../entities/vhost.md), providing low-latency interrupt delivery that bypasses QEMU's event loop entirely.

## SR-IOV Passthrough

SR-IOV (Single Root I/O Virtualization) extends the passthrough model to allow **multiple VMs to share a single physical device** while maintaining hardware-level isolation.

### Physical Functions and Virtual Functions

A SR-IOV capable device exposes:

- **Physical Function (PF)**: The full-featured PCIe function. The host driver manages the PF to configure the device and create VFs.
- **Virtual Functions (VF)**: Lightweight PCIe functions, each with its own BARs, MSI-X vectors, and queue pairs. Each VF appears as an independent PCI device that can be assigned to a VM via VFIO.

```
┌────────────────────────────────────┐
│        Physical NIC                │
│  ┌──────┐  ┌────┐ ┌────┐ ┌────┐  │
│  │  PF  │  │ VF0│ │ VF1│ │ VF2│  │
│  │(host)│  │(VM1)│ │(VM2)│ │(VM3)│ │
│  └──────┘  └────┘ └────┘ └────┘  │
│        Hardware packet steering    │
└────────────────────────────────────┘
```

The hardware steers packets to the correct VF based on MAC address or VLAN, and each VF has dedicated queue resources. There is no software switching overhead.

### VF Creation and Assignment

```bash
# Check SR-IOV capability
cat /sys/class/net/eth0/device/sriov_totalvfs    # e.g., 63

# Create VFs
echo 4 > /sys/class/net/eth0/device/sriov_numvfs

# Each VF appears as a new PCI device
lspci | grep "Virtual Function"

# Bind VF to vfio-pci and assign to VM (same process as any device)
```

See [kvm-networking](../entities/kvm-networking.md) for libvirt XML examples of VF assignment using `<hostdev>` and `<interface type='hostdev'>`.

### SR-IOV vs Full Device Passthrough

| Aspect | Full device passthrough | SR-IOV VF passthrough |
|--------|------------------------|----------------------|
| Devices per physical NIC | 1 VM per NIC | Many VMs per NIC (up to 256 VFs) |
| Performance | Full device bandwidth | Per-VF bandwidth (hardware QoS) |
| Host driver | Unloaded (device unavailable to host) | PF remains under host control |
| Management | Device invisible to host | Host PF driver manages VF creation |

## Security Model

VFIO's security rests on three pillars:

### 1. DMA Isolation

The IOMMU page tables ensure a device can only DMA to memory explicitly mapped for its VFIO container. A malicious device (or a compromised guest driving a real device) cannot read or write host memory, other guests' memory, or QEMU's address space beyond the mapped regions.

### 2. Interrupt Isolation

Interrupt remapping prevents a device from injecting interrupts to arbitrary CPUs or vectors. Without interrupt remapping, a device could write crafted MSI messages to target any interrupt vector on any CPU — potentially injecting interrupts into other VMs or the hypervisor.

### 3. Group-Level Granularity

VFIO enforces that all devices in an IOMMU group must be controlled by VFIO (or a known-safe driver) before any device in the group can be assigned. This prevents a scenario where Device A is assigned to a VM while Device B in the same group (still under a host driver) could be used as a proxy to attack the VM's IOMMU domain via peer-to-peer DMA.

## Trade-offs and Limitations

**Advantages:**
- Near-native I/O performance: no emulation, no paravirtual overhead, no hypervisor in the data path
- Full device feature set exposed to guest (hardware offloads, proprietary features)
- Reduced attack surface: less QEMU device emulation code in the I/O path

**Limitations:**
- **No live migration**: device state resides in hardware registers and internal ASIC state that cannot be captured and transferred. The guest must be shut down or the device hot-unplugged before migration. This is the primary operational trade-off vs virtio.
- **Reduced device sharing**: without SR-IOV, one physical device serves one VM. Even with SR-IOV, the number of VFs is hardware-limited.
- **Host loses device access**: the device is unavailable to the host OS while assigned to a VM.
- **Hardware dependency**: requires IOMMU-capable hardware (VT-d / AMD-Vi), IOMMU-capable motherboard/BIOS, and ACS-capable PCIe topology for fine-grained isolation.
- **IOMMU group constraints**: multi-function devices or devices behind non-ACS switches force group-level assignment, potentially requiring more devices to be passed through than desired.

## See also

- [device-driver-model](../entities/device-driver-model.md)
- [kvm-networking](../entities/kvm-networking.md)
- [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md)
- [kvm-memory-virtualization](../entities/kvm-memory-virtualization.md)
- [kvm-performance-tuning](../entities/kvm-performance-tuning.md)
- [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md)
- [concept-virtio-data-plane](../concepts/concept-virtio-data-plane.md)
- [vfio-device-passthrough-zh](vfio-device-passthrough-zh.md) — Chinese version / 中文版
