---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [qemu-kvm-source-code-and-application]
tags: [kvm, interrupt-virtualization, apicv, posted-interrupts, pic, ioapic, lapic, msi]
---

# KVM Interrupt Virtualization

Interrupt virtualization is one of the most complex aspects of KVM's x86 platform emulation. KVM must faithfully reproduce the behavior of hardware interrupt controllers—PIC, I/O APIC, and Local APIC—so that guest operating systems can manage interrupts without modification. Modern hardware extensions (APICv, posted interrupts) allow much of this to happen without costly VM Exits.

This page covers interrupt controller emulation in QEMU/KVM based on QEMU 2.8.1 and kernel 4.4.161.

## Interrupt Controller Types

x86 platforms have three generations of interrupt controllers, all of which KVM emulates in-kernel.

### PIC (8259A)

The legacy Programmable Interrupt Controller uses two cascaded 8259A chips. The slave PIC connects to IRQ 2 of the master, yielding 15 usable IRQ lines (IRQ 0–7 on the master, IRQ 8–15 on the slave). Each 8259A maintains three key registers:

- **IRR** (Interrupt Request Register) — bits set when an interrupt is asserted.
- **ISR** (In-Service Register) — bits set when an interrupt is being serviced by the CPU.
- **IMR** (Interrupt Mask Register) — bits set to mask (disable) specific IRQ lines.

### Local APIC

Each logical CPU has a dedicated Local APIC (LAPIC). It handles four categories of interrupts:

1. **Local interrupts** — LINT0, LINT1, performance counters, thermal sensor, APIC error.
2. **Timer interrupts** — the APIC timer with one-shot, periodic, and TSC-deadline modes.
3. **Inter-Processor Interrupts (IPIs)** — sent between CPUs for TLB shootdowns, rescheduling, etc.
4. **External interrupts** — routed from the I/O APIC or delivered via MSI.

The LAPIC register space is memory-mapped at a configurable base address (default `0xFEE00000`), occupying one 4 KB page.

### I/O APIC

The I/O APIC is a system-level interrupt router that replaces the PIC for multiprocessor configurations. It has a 24-entry redirection table, where each entry maps a physical interrupt pin to a destination LAPIC and vector. This allows flexible routing of device interrupts across CPUs.

## GSI Routing

KVM uses a Global System Interrupt (GSI) abstraction to route interrupts from devices to the correct interrupt controller. The key structures are:

- **GSIState** — tracks the combined interrupt level for each GSI from all asserting sources.
- **kvm_pc_gsi_handler()** — the default GSI handler for PC machines. For GSI 0–15 it routes to the PIC; for GSI 0–23 it routes to the I/O APIC.
- **kvm_irq_routing_table** — an in-kernel table that maps each GSI number to one or more routing entries. Each entry specifies the controller type (PIC, I/O APIC, or MSI) and the parameters needed to deliver the interrupt.

The routing table is populated by QEMU at VM setup via the `KVM_SET_GSI_ROUTING` ioctl and can be updated dynamically as devices are configured.

## PIC Emulation

KVM emulates the dual-8259A PIC entirely in-kernel. The key functions are:

- **kvm_pic_set_irq()** — entry point when a device asserts or deasserts an IRQ line. Determines which chip (master or slave) handles the IRQ and delegates to `pic_set_irq1()`.
- **pic_set_irq1()** — updates the IRR for the specified chip. For edge-triggered interrupts, it sets the IRR bit on a 0→1 transition. For level-triggered interrupts, it sets or clears the IRR bit based on the current level.
- **pic_update_irq()** — scans IRR & ~IMR to find the highest-priority pending interrupt. If the slave has a pending interrupt, it cascades through IRQ 2 on the master. Sets an output IRQ flag indicating an interrupt is ready.

When a VCPU is about to enter the guest, KVM calls **kvm_cpu_get_interrupt()** to retrieve the vector number from the PIC. This function acknowledges the interrupt (moves the bit from IRR to ISR) and returns the vector for injection into the guest via the VMCS/VMCB.

## I/O APIC Emulation

The in-kernel I/O APIC is represented by the **kvm_ioapic** struct, which contains 24 redirection table entries. Each entry (**kvm_ioapic_redirect_entry**) has these fields:

| Field | Purpose |
|-------|---------|
| `vector` | Interrupt vector number (0–255) |
| `delivery_mode` | Fixed, lowest-priority, SMI, NMI, INIT, or ExtINT |
| `dest_mode` | Physical (target by APIC ID) or logical (target by logical ID bitmask) |
| `trig_mode` | Edge-triggered or level-triggered |
| `mask` | If set, the interrupt is masked and will not be delivered |
| `dest_id` | Destination APIC ID or logical destination |

The delivery path is:

1. **ioapic_set_irq()** — called when a device asserts a GSI routed to the I/O APIC. Looks up the redirection table entry for the pin. If the entry is masked, the interrupt is recorded but not delivered.
2. **ioapic_service()** — formats the interrupt according to the redirection entry and calls the LAPIC delivery function.
3. **kvm_irq_delivery_to_apic()** — iterates over all VCPUs to find matching destination LAPICs (based on `dest_mode` and `dest_id`), then calls `kvm_apic_set_irq()` on each matching LAPIC.

For level-triggered interrupts, the I/O APIC tracks the remote IRR bit. The interrupt pin remains asserted until the guest issues an EOI, which the LAPIC forwards back to the I/O APIC to clear the remote IRR.

## Local APIC Emulation

The in-kernel LAPIC is represented by the **kvm_lapic** struct, which contains:

- **base_address** — the MMIO base address of the APIC register space.
- **regs** — a 4 KB page holding all APIC registers (ID, version, TPR, APR, PPR, EOI, LDR, DFR, SVR, ISR, TMR, IRR, ESR, ICR, LVT entries, timer registers, etc.).

### Register Access Emulation

When the guest reads or writes APIC registers (and hardware APIC virtualization is not active), a VM Exit occurs. KVM handles the exit by dispatching to register-specific read/write handlers. Writes to the ICR trigger IPI delivery; writes to the EOI register trigger end-of-interrupt processing; writes to LVT entries configure local interrupt sources.

### APIC Timer

The LAPIC timer is implemented using the host kernel's **hrtimer** infrastructure. KVM supports three timer modes:

- **One-shot** — fires once after the initial count decrements to zero.
- **Periodic** — fires repeatedly at the programmed interval.
- **TSC-deadline** — fires when the TSC reaches the value written to the `IA32_TSC_DEADLINE` MSR. This is the preferred mode for modern guests due to its precision.

When the hrtimer fires, KVM calls **kvm_apic_local_deliver()** to inject the timer interrupt into the guest's LAPIC IRR using the vector from the LVT timer entry.

### Local Interrupt Delivery

**kvm_apic_local_deliver()** handles delivery of local interrupt sources (timer, LINT0, LINT1, performance counter, thermal, error). It reads the corresponding LVT entry, checks the mask bit, and if unmasked, sets the appropriate bit in the IRR.

### Interrupt Injection at VM Entry

Before each VM Entry, KVM checks whether the guest LAPIC has any pending interrupts (bits set in IRR & ~ISR with priority higher than PPR). If so, KVM:

1. Finds the highest-priority pending vector.
2. Moves the vector's bit from IRR to ISR.
3. Writes the vector into the VMCS VM-entry interruption-information field.
4. Executes VMLAUNCH/VMRESUME, and the CPU delivers the interrupt to the guest immediately upon entry.

## MSI (Message Signaled Interrupts)

MSI bypasses the I/O APIC entirely. Instead of asserting a physical pin, the device performs a memory write to a special address that the LAPIC intercepts. This provides several advantages:

- No pin-sharing conflicts.
- Lower latency (no I/O APIC routing step).
- Support for more interrupt vectors per device (up to 32 for MSI, 2048 for MSI-X).

In QEMU/KVM, MSI delivery follows this path:

1. **msi_notify()** — called when a device triggers an MSI. Calls `msi_get_message()` to retrieve the MSI address and data.
2. **msi_get_message()** — reads the device's MSI capability registers and constructs an **MSIMessage** struct containing the `address` (encodes destination APIC ID and delivery mode) and `data` (encodes the vector number).
3. **kvm_set_msi()** — sends the MSI message to KVM via the **KVM_SIGNAL_MSI** ioctl. KVM decodes the address and data fields and delivers the interrupt directly to the target LAPIC, exactly as if the I/O APIC had routed it.

## APICv (Hardware APIC Virtualization)

APICv is Intel's hardware acceleration for APIC operations, eliminating VM Exits for most interrupt management tasks. AMD provides equivalent features under the name AVIC.

### VM-Execution Controls

APICv is enabled through several VMCS VM-execution control bits:

| Control | Effect |
|---------|--------|
| **Virtualize APIC accesses** | Traps guest accesses to the APIC MMIO page and redirects them to the virtual-APIC page. |
| **APIC-register virtualization** | Allows guest reads of most APIC registers to complete without a VM Exit, served from the virtual-APIC page. |
| **TPR shadow** | Guest reads/writes to the Task Priority Register (CR8 or APIC TPR) do not cause VM Exits. |
| **Virtual-interrupt delivery** | The processor evaluates and delivers pending virtual interrupts to the guest automatically, without a VM Exit. |
| **Process posted interrupts** | The processor processes a posted-interrupt notification and merges posted IRQs into the virtual-APIC page without a VM Exit. |

### Key Data Structures

- **Virtual-APIC page** — a per-VCPU 4 KB page that shadows the APIC register space. The processor reads and writes this page instead of causing VM Exits for APIC register accesses.
- **APIC-access page** — a single shared 4 KB page mapped at the physical address `0xFEE00000` (the standard APIC base). Guest accesses to this address are intercepted and redirected to the per-VCPU virtual-APIC page.
- **Guest Interrupt Status** — a 16-bit field in the VMCS containing:
  - **RVI** (Requesting Virtual Interrupt) — the highest-priority pending virtual interrupt vector.
  - **SVI** (Servicing Virtual Interrupt) — the vector of the interrupt currently being serviced.

### Posted-Interrupt Descriptor

The posted-interrupt descriptor (**pi_desc**) is a per-VCPU data structure used by the processor for posted-interrupt processing:

| Field | Size | Purpose |
|-------|------|---------|
| `pir[8]` | 256 bits | Posted Interrupt Requests bitmap — one bit per vector. |
| `on` | 1 bit | Outstanding Notification — set when new interrupts have been posted. |
| `sn` | 1 bit | Suppress Notification — when set, the processor does not send a notification IPI. |
| `nv` | 8 bits | Notification Vector — the vector used for the posted-interrupt notification IPI. |
| `ndst` | 32 bits | Notification Destination — the physical APIC ID of the VCPU to notify. |

### Delivery Flow

When KVM needs to deliver an interrupt to a VCPU with APICv enabled:

1. **vmx_deliver_posted_interrupt()** — sets the appropriate bit in `pir[]` and atomically sets the `on` (outstanding notification) flag.
2. If the target VCPU is currently running in guest mode, KVM sends an IPI with vector `POSTED_INTR_VECTOR` to the physical CPU hosting that VCPU.
3. The physical CPU receives the notification IPI, recognizes it as a posted-interrupt notification, and automatically:
   - Clears the `on` bit.
   - Copies bits from `pir[]` into the virtual-APIC page's VIRR (Virtual IRR).
   - Updates RVI in the Guest Interrupt Status.
   - Delivers the highest-priority pending interrupt to the guest — all without a VM Exit.
4. If the VCPU is not in guest mode, the interrupt is processed at the next VM Entry via **vmx_sync_pir_to_irr()**, which copies pending bits from `pir[]` to IRR in the virtual-APIC page before executing VMLAUNCH/VMRESUME.

This mechanism is particularly valuable for high-throughput I/O workloads where devices (especially [virtio-framework](virtio-framework.md) devices via [vhost](vhost.md)) generate frequent interrupts.

## IRQfd

IRQfd provides a mechanism for injecting interrupts into a KVM guest via eventfd file descriptors. It is configured through the **KVM_IRQFD** ioctl.

**kvm_irqfd_assign()** sets up the association:

1. Creates or reuses an eventfd.
2. Associates the eventfd with a specific GSI number.
3. Registers a polling callback on the eventfd.

When the eventfd is signaled (e.g., by a [vhost](vhost.md) backend writing to it), the polling callback fires in the host kernel and calls **kvm_set_irq()** to inject the interrupt through the standard GSI routing path.

IRQfd is essential for efficient interrupt delivery from kernel-based backends. Without IRQfd, interrupt injection would require a round-trip through QEMU userspace, adding significant latency.

## IOeventfd

IOeventfd is the complementary mechanism to IRQfd. Configured through the **KVM_IOEVENTFD** ioctl, it associates an eventfd with a specific I/O port or MMIO address.

When the guest writes to the registered address:

1. KVM intercepts the write (via VM Exit or EPT violation).
2. KVM matches the address against registered ioeventfds.
3. If a match is found, KVM signals the associated eventfd **and returns to the guest immediately** — no exit to QEMU userspace is needed.
4. The eventfd signal notifies the backend (e.g., a vhost worker thread) that the guest has performed a doorbell write.

IOeventfd is heavily used for [virtio-framework](virtio-framework.md) doorbell notifications (virtqueue kick). The guest writes to a specific MMIO or PIO address to notify the device that new buffers are available, and the eventfd efficiently routes this notification to the host backend without involving QEMU's main event loop.

Together, IRQfd and IOeventfd form the fast-path interrupt and notification mechanism that makes [vhost](vhost.md) possible, keeping both interrupt injection and doorbell notifications entirely within the host kernel.

## Interrupt Delivery Summary

The following summarizes the path from device to guest for each mechanism:

- **PIC path**: Device → `kvm_pic_set_irq()` → IRR/ISR → `kvm_cpu_get_interrupt()` → VMCS injection.
- **I/O APIC path**: Device → `ioapic_set_irq()` → `ioapic_service()` → `kvm_irq_delivery_to_apic()` → LAPIC IRR → VMCS injection.
- **MSI path**: Device → `msi_notify()` → `kvm_set_msi()` → LAPIC IRR → VMCS injection.
- **APICv path**: Device → `vmx_deliver_posted_interrupt()` → `pi_desc.pir[]` → hardware merges to VIRR → automatic guest delivery (no VM Exit).
- **IRQfd path**: eventfd signal → `kvm_set_irq()` → GSI routing → appropriate controller → guest delivery.

## See also

- [kvm-cpu-virtualization](kvm-cpu-virtualization.md)
- [virtio-framework](virtio-framework.md)
- [vhost](vhost.md)
- [interrupt-handling](interrupt-handling.md)
