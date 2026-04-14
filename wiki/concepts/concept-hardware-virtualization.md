---
type: concept
created: 2026-04-09
updated: 2026-04-09
sources: [qemu-kvm-source-code-and-application]
tags: [virtualization, vmx, vt-x, hypervisor, trap-and-emulate]
---

# Hardware Virtualization

> "Any problem in computer science can be solved by adding another level of indirection."
> -- David Wheeler

This principle underlies virtually every abstraction in computing, and virtualization is its most literal embodiment. Processes virtualize the CPU and memory for user programs. Emulators virtualize one instruction set atop another. Language VMs like the JVM virtualize a machine entirely in software. System-level virtual machines virtualize an entire hardware platform so that unmodified operating systems can run as guests. The history of virtualization is a history of refining that indirection until its overhead approaches zero.

## Historical Context

**1960s -- IBM mainframes.** IBM's CP-40 and CP-67 introduced the concept of virtual machines on System/360 hardware. Each user got what appeared to be a dedicated mainframe. The key insight was that the hardware itself could enforce isolation between virtual machines through privileged instruction trapping.

**1998 -- VMware Workstation.** x86 was notoriously difficult to virtualize because certain privileged instructions failed silently in user mode rather than trapping. VMware solved this with binary translation: scanning guest code at runtime and rewriting problematic instructions.

**2001 -- Xen.** The University of Cambridge took a different approach with paravirtualization: modify the guest OS kernel to replace un-virtualizable instructions with explicit hypervisor calls (hypercalls). This avoided binary translation but required guest cooperation.

**2005-2006 -- Hardware-assisted virtualization.** Intel VT-x and AMD-V added new CPU modes and instructions purpose-built for virtualization, eliminating the need for binary translation or guest modification.

**2006 -- KVM.** Avi Kivity submitted KVM as a Linux kernel module, turning Linux itself into a hypervisor. By leveraging hardware-assisted virtualization, KVM's initial implementation was remarkably small (around 10,000 lines of code). Combined with QEMU for device emulation, KVM became a dominant platform for cloud virtualization.

## Hypervisor Taxonomy

### Type 1: Bare-Metal Hypervisors

A Type 1 hypervisor runs directly on hardware with no host OS beneath it. The hypervisor itself is the most privileged software layer. Examples include VMware ESXi, Xen, and Microsoft Hyper-V.

Characteristics:
- Direct hardware access with minimal overhead
- The hypervisor manages hardware resources and scheduling
- Guest OSes run in less privileged modes managed by the hypervisor
- Typically used in data centers and production environments

### Type 2: Hosted Hypervisors

A Type 2 hypervisor runs as an application on top of a conventional host OS. The host kernel retains full hardware control, and the hypervisor uses host OS services for resource management. Examples include VMware Workstation, VirtualBox, and standalone QEMU.

Characteristics:
- Easier to install and use (just another application)
- Additional overhead from the host OS layer
- Relies on the host OS for hardware drivers and scheduling
- Typically used for development and testing

### QEMU/KVM: Blurring the Line

QEMU by itself is a Type 2 hypervisor -- it runs as a userspace process and emulates hardware in software. But when paired with KVM, the classification becomes interesting. KVM is a kernel module that turns the Linux kernel itself into a bare-metal hypervisor. Guest code executes directly on the CPU in a hardware-isolated mode (VMX non-root). In this configuration, QEMU acts as the userspace management component of what is effectively a Type 1 hypervisor. The Linux kernel, with KVM loaded, is the hypervisor.

## Full Virtualization vs Paravirtualization

### Full Virtualization

Full virtualization presents the guest with a complete emulation of the underlying hardware. The guest OS runs unmodified -- it has no idea it is inside a virtual machine.

Two approaches exist:

1. **Binary translation** (pre-hardware-assist era): The hypervisor scans guest kernel code, identifies sensitive instructions that would not trap properly on x86, and rewrites them to safe equivalents at runtime. VMware pioneered this approach.

2. **Hardware-assisted virtualization**: The CPU provides a new execution mode where sensitive instructions automatically trap to the hypervisor. No binary translation or guest modification needed.

### Paravirtualization

Paravirtualization modifies the guest OS to be aware that it is running inside a virtual machine. Instead of executing privileged instructions that would need to be trapped and emulated, the guest explicitly calls into the hypervisor via hypercalls (analogous to system calls, but from guest kernel to hypervisor).

Xen popularized this approach. The guest replaces hardware-specific operations (page table updates, interrupt handling, I/O) with hypercalls. This avoids the overhead of trapping and emulating privileged instructions but requires maintaining a modified guest kernel.

### Virtio: Paravirtualized I/O

Even with hardware-assisted CPU virtualization, I/O device emulation remains a bottleneck. Virtio provides a standardized paravirtualized I/O framework: the guest runs virtio drivers that know they are in a VM and communicate with the hypervisor through shared memory ring buffers (VRings) rather than emulating real hardware register accesses. This dramatically reduces the number of VM Exits required for I/O operations. See [virtio-framework](../entities/virtio-framework.md) for details.

## Hardware-Assisted Virtualization

### Intel VT-x (VMX)

Intel VT-x introduces Virtual Machine Extensions (VMX), a set of new CPU instructions and execution modes designed specifically for virtualization.

**VMX Modes:**

The CPU operates in one of two modes:

- **VMX root mode**: Where the hypervisor (KVM) runs. This mode has full privilege and behaves largely like normal x86 execution. It has its own ring 0 through ring 3.
- **VMX non-root mode**: Where the guest OS and its applications run. This mode also has ring 0 through ring 3, so the guest kernel runs at ring 0 of non-root mode. However, certain operations automatically cause a VM Exit, transferring control back to VMX root mode.

This two-dimensional privilege model (root/non-root x ring 0-3) is the key innovation. The guest kernel genuinely runs at ring 0, retaining full access to its own privilege level, but the hardware ensures that sensitive operations trap to the hypervisor transparently.

**Key VMX Instructions:**

| Instruction | Purpose |
|-------------|---------|
| `VMXON` | Enable VMX operation (enter VMX root mode) |
| `VMXOFF` | Disable VMX operation |
| `VMLAUNCH` | Launch a VM for the first time (enter non-root mode) |
| `VMRESUME` | Resume a previously launched VM |
| `VMREAD`/`VMWRITE` | Read/write fields of the VMCS |
| `VMPTRLD`/`VMPTRST` | Load/store pointer to VMCS |

**VMCS (Virtual Machine Control Structure):**

The VMCS is a hardware-defined data structure that controls VM Exits and VM Entries. It contains guest state (registers, segment descriptors), host state (where to resume after a VM Exit), and control fields specifying which guest operations should cause VM Exits. KVM configures the VMCS to trap exactly the operations it needs to mediate.

### AMD-V (SVM)

AMD's virtualization extensions are called AMD-V, with the instruction set known as Secure Virtual Machine (SVM). The concepts mirror VT-x:

- `VMRUN` instruction launches or resumes a guest (equivalent to VMLAUNCH/VMRESUME)
- `VMEXIT` returns control to the hypervisor
- VMCB (Virtual Machine Control Block) serves the same role as Intel's VMCS
- Guest and host modes correspond to VMX non-root and root

While the instruction mnemonics and data structure layouts differ, the architectural principles are identical: hardware-enforced isolation of guest execution with configurable trapping of sensitive operations.

## Progressive Elimination of VM Exits

VM Exits are expensive. Each exit requires saving full guest state, loading host state, executing hypervisor logic, then reversing the process on VM Entry. A VM Exit can cost hundreds to thousands of CPU cycles. The evolution of hardware virtualization has been a systematic effort to eliminate unnecessary VM Exits across three domains:

### Memory: EPT / NPT

**Problem:** Guest operating systems manage their own page tables, but those page tables contain guest-physical addresses, not host-physical addresses. Without hardware support, every guest page table update requires a VM Exit so the hypervisor can maintain shadow page tables that map guest-virtual to host-physical addresses.

**Solution:** Extended Page Tables (Intel EPT) and Nested Page Tables (AMD NPT) add a second level of address translation in hardware. The CPU walks the guest page table to get a guest-physical address, then automatically walks the EPT/NPT to translate that to a host-physical address. Guest page table operations proceed without VM Exits. See [kvm-memory-virtualization](../entities/kvm-memory-virtualization.md).

### Interrupts: APICv / AVIC

**Problem:** Interrupt delivery to a guest normally requires a VM Exit: the hypervisor intercepts the interrupt, determines the target vCPU, and injects it during VM Entry.

**Solution:** Intel APICv and AMD AVIC allow hardware to deliver certain interrupts directly into the guest without a VM Exit. The hypervisor sets up data structures that the hardware consults, enabling posted interrupts to be injected while the guest is running. See [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md).

### I/O: VT-d (IOMMU)

**Problem:** I/O device access from a guest requires the hypervisor to mediate every interaction -- either emulating a virtual device or forwarding operations to a physical device.

**Solution:** Intel VT-d (and AMD's equivalent IOMMU technology) enables device passthrough. A physical device can be assigned directly to a guest VM. The IOMMU translates device DMA addresses using guest-physical-to-host-physical mappings, and the device's interrupts are routed directly to the guest. The hypervisor is removed from the I/O data path entirely.

### The Trajectory

Each generation of hardware virtualization extensions eliminates another category of VM Exits:

```
Generation 1: VT-x/AMD-V     -- CPU virtualization (eliminate binary translation)
Generation 2: EPT/NPT         -- Memory virtualization (eliminate shadow page tables)
Generation 3: APICv/AVIC      -- Interrupt virtualization (eliminate interrupt interception)
Generation 4: VT-d/IOMMU      -- I/O virtualization (eliminate device emulation)
```

The end state approaches bare-metal performance: the guest runs on the CPU with hardware-enforced isolation, hardware-translated memory, hardware-delivered interrupts, and direct hardware device access. The hypervisor only intervenes for management operations and resource allocation, not for the data path.

## See also

- [qemu-kvm-overview](../entities/qemu-kvm-overview.md)
- [kvm-cpu-virtualization](../entities/kvm-cpu-virtualization.md)
- [kvm-memory-virtualization](../entities/kvm-memory-virtualization.md)
- [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md)
- [vfio-device-passthrough](../analyses/vfio-device-passthrough.md)
