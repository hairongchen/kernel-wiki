---
type: source
created: 2026-04-10
updated: 2026-04-10
title: "Improvement of Real-time Performance of KVM"
sources: [lcna-co2012-sekiyama]
tags: [kvm, real-time, cpu-isolation, direct-interrupt-delivery, direct-eoi, virtualization]
---

# Source: Improvement of Real-time Performance of KVM

**Author:** Tomoki Sekiyama, Linux Technology Center, Yokohama Research Lab, Hitachi, Ltd.
**Venue:** LinuxCon North America 2012
**Format:** Presentation slides (20 slides)
**Coverage:** Real-time virtualization on KVM — CPU isolation, direct interrupt delivery, and direct EOI techniques to eliminate VM Exit overhead for latency-sensitive workloads.

## Summary

This presentation proposes a set of KVM modifications to achieve near-bare-metal real-time performance for virtualized guests. The core insight is that by **dedicating physical CPUs** to guest VMs and **bypassing the hypervisor's interrupt interception**, most VM Exit overhead on the interrupt path can be eliminated. The work targets industrial control systems, embedded appliances, enterprise trading systems, and HPC workloads.

### Section 1: Overview of Realtime Virtualization (Slides 1-7)

**Use cases for RT virtualization:**
- Factory automation / social infrastructure control — low latency, deadline constraints, typically single-core, long lifecycle (10+ years)
- Embedded systems / appliances — run RTOS guest alongside Linux for gradual migration
- Enterprise systems (automated trading) — preserve legacy software on new hardware
- HPC — similar latency requirements, cloud deployment (e.g., Amazon EC2)

**Requirements:**
- Low latency (fast external event response)
- Bare-metal performance (no application slowdown)
- Preserve at least soft real-time quality (no temporal interference from host)
- Sometimes guest OS modification must be avoided (legacy GPOS/RTOS)

**Why KVM:** Advanced features (EPT, x2APIC, VT-d), well-defined management interfaces (libvirt), large community, upstreamed in Linux kernel.

**Issues in RT virtualization:**
- `mlock(2)`, `SCHED_RR`, and exclusive cpuset help but are insufficient
- Host kernel thread interference remains
- Interrupt forwarding overhead is the primary bottleneck — each timer interrupt involves 3 VM Exit/Entry pairs (set APIC timer → emulate → timer fires → vIRQ injection → EOI → emulate APIC access)
- Passed-through PCI device interrupts follow the same expensive path

### Section 2: Improvement of KVM Realtime Performance (Slides 8-20)

**Two-pronged approach:**

#### CPU Isolation (Slides 9-11)

Dedicate physical CPUs to the guest by taking them offline from the Linux host:

1. Offline the CPU: `echo 0 > /sys/devices/system/cpu/cpuX/online`
2. Bind vCPU to the dedicated CPU via a new ioctl: `ioctl(vcpu[i], KVM_SET_SLAVE_CPU, slave_cpu_id[i])`
3. The CPU boots with minimal hypervisor function — only runs the vCPU, no host kernel threads
4. vCPU thread on the host side suspends while guest runs on the dedicated CPU (resumes only on VM Exit requiring QEMU)

**Benefits:**
- Eliminates interference from host kernel tasks
- Assures bare-metal CPU performance — no interruption by other guests or host processes
- Enables guest to occupy CPU facilities (local APIC, etc.) — prerequisite for direct IRQ delivery

#### Direct Interrupt Delivery (Slides 12-17)

Exploits VT-x/AMD SVM hardware to deliver interrupts directly to the guest IDT, bypassing VM Exit:

**Core mechanism:** Clear the "External interrupt exiting" bit (bit 0) in the VMCS Pin-Based VM-Execution Controls. When this bit is 0, external interrupts are delivered directly through the guest IDT instead of causing VM Exits.

**Issue #1 — Cannot distinguish host vs. guest interrupts:**
- With external interrupt exiting disabled, ALL interrupts on that CPU go to the guest
- Solution: CPU isolation + IRQ affinity — set IRQ affinity so host device interrupts go to host cores, passed-through device interrupts go to dedicated guest cores
- Currently only MSI/MSI-X supported (per-vector affinity); shared ISA IRQs still require host forwarding

**Issue #2 — Cannot send normal IPI to dedicated CPUs:**
- Host needs to send IPIs to dedicated CPUs for virtual IRQ injection, TLB shootdowns, etc.
- But with external interrupt exiting disabled, IPIs would be delivered to the guest
- Solution: Use NMI instead of normal IPI — NMI exiting can be controlled independently in VMCS. NMI is used purely to cause a VM Exit, after which the host checks requests from other CPUs

**Issue #3 — Host and guest use different interrupt vectors:**
- Normal KVM converts host vector to guest vector during interrupt forwarding
- With direct delivery, PCI devices must be reconfigured with the guest's vector numbers
- Problem: host receives the guest's vector during VM Exit periods (I/O emulation)
- Solution: Register the guest's vector→IRQ mapping on the dedicated CPU's IDT, so if host receives the guest vector, it injects it as vIRQ

#### Direct EOI (Slides 19-20)

With x2APIC hardware, the guest can perform End-Of-Interrupt directly via MSR access:

- x2APIC provides APIC access via MSRs (Model Specific Registers)
- VT-x MSR bitmap controls which MSRs are exposed to the guest — expose the EOI MSR for direct access
- **Constraint:** Direct EOI must NOT be applied to virtual IRQs — EOI for virtual IRQs must go to the emulated virtual APIC
- Solution: Disable direct EOI when injecting a virtual IRQ; re-enable after every virtual IRQ is handled

#### Interrupt Flow Comparison

**Normal KVM:** External interrupt → VM Exit → host IRQ handler → vIRQ injection → VM Enter → guest IRQ handler → EOI → VM Exit → emulate APIC access → VM Enter (3 VM Exit/Entry pairs)

**Direct delivery + direct EOI:** External interrupt → guest IRQ handler → direct EOI (0 VM Exits for passed-through device interrupts)

**Virtual interrupt with direct delivery:** NMI → VM Exit → QEMU device emulation → vIRQ injection → VM Enter → guest IRQ handler → EOI → VM Exit → emulate APIC access → VM Enter → re-enable direct EOI (2 VM Exits, but only for emulated device interrupts)

## Key Themes

- **CPU partitioning as a prerequisite** — direct interrupt delivery only works when CPUs are fully dedicated to guests, making it unsuitable for overcommitted cloud environments but ideal for RT/embedded scenarios
- **Leveraging existing hardware** — the approach uses VT-x features (Pin-Based controls, MSR bitmaps, NMI exiting) that already existed in 2012 hardware, requiring no new CPU features
- **Trade-off: isolation vs. flexibility** — achieving bare-metal interrupt performance requires giving up CPU overcommitment and host-side interrupt multiplexing

## Relationship to Other Sources

- Complements [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md) — shows an alternative to APICv for interrupt VM Exit elimination via CPU isolation + direct delivery
- Relates to [kvm-performance-tuning](../entities/kvm-performance-tuning.md) — CPU isolation is an extreme form of CPU pinning
- Extends [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md) — demonstrates real-time use cases driving VM Exit elimination
- Contrast with APICv (later hardware generation) which achieves similar goals without requiring CPU dedication

## See also

- [analysis-direct-interrupt-delivery-sekiyama](../analyses/analysis-direct-interrupt-delivery-sekiyama.md)
- [src-vmexit-opt-hitachi-sekiyama](src-vmexit-opt-hitachi-sekiyama.md)
- [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md)
- [kvm-cpu-virtualization](../entities/kvm-cpu-virtualization.md)
- [kvm-performance-tuning](../entities/kvm-performance-tuning.md)
- [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md)
- [analysis-vm-exit-reduction-and-timer-virtualization](../analyses/analysis-vm-exit-reduction-and-timer-virtualization.md)
