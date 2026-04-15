---
type: source
created: 2026-04-11
updated: 2026-04-11
title: "更早的替代方案：直接中断投递（Sekiyama, 2012）"
sources: [vmexit-opt-hitachi-sekiyama]
tags: [kvm, direct-interrupt-delivery, cpu-isolation, vm-exit, apicv, real-time, virtualization]
---

# Source: 直接中断投递分析（Sekiyama 方案解读）

**Author:** Anonymous (Chinese-language technical analysis)
**Subject:** Tomoki Sekiyama's direct interrupt delivery approach (Hitachi, LinuxCon 2012)
**Format:** Markdown article (Chinese)
**Coverage:** Step-by-step technical breakdown of the Sekiyama interrupt VM Exit elimination scheme, with pros/cons analysis and comparison to modern APICv.

## Summary

This article provides a detailed Chinese-language analysis of the direct interrupt delivery technique proposed by Tomoki Sekiyama (Hitachi) at LinuxCon 2012. It complements the [original presentation source](src-lcna-co2012-sekiyama.md) by reorganizing the material into a six-step construction of the "direct channel" and adding an explicit advantages/disadvantages evaluation.

### Section 1: Background and Core Problem

Describes the traditional KVM interrupt path: hardware interrupt → VM Exit (due to VMCS "External interrupt exiting" bit) → Host KVM parses and injects → Guest handles. The performance bottleneck is the VM Exit/VM Entry context switch (register save/restore, TLB flush) on every interrupt, particularly punishing for network I/O intensive workloads.

### Section 2: Six-Step Technical Construction

The article breaks the solution into six clearly labeled steps:

**A. CPU Isolation** — Take the target pCPU offline via `echo 0 > /sys/devices/system/cpu/cpuX/online` to remove it from the host scheduler entirely.

**B. Dedicated CPU Binding (KVM_SET_SLAVE_CPU)** — A Hitachi-specific KVM kernel patch (not mainline). The `ioctl(KVM_SET_SLAVE_CPU)` establishes a **1:1 vCPU-to-pCPU exclusive mapping**, preventing migration and providing the physical foundation for direct interrupt delivery.

**C. Disable External Interrupt Exiting (the "core magic")** — Clear bit 0 of the VMCS Pin-Based Controls. This changes behavior from "interrupt → VM Exit → Host → inject → Guest" to "interrupt → CPU directly jumps to Guest IDT handler". The host loses its ability to intercept ordinary interrupts on that CPU.

**D. IRQ Routing** — Use `smp_affinity` or `irqbalance` blacklisting to route host device interrupts to non-isolated CPUs, and passthrough device MSI/MSI-X interrupts exclusively to the isolated dedicated CPU.

**E. Host Communication via NMI** — Since ordinary IPIs can no longer reach the host on isolated CPUs, the host uses NMI (Non-Maskable Interrupt) instead. NMI exiting is controlled by a separate VMCS bit and can still force a VM Exit. KVM registers an NMI handler for special host management requests (TLB shootdown, reschedule).

**F. Direct EOI via x2APIC Passthrough** — Expose the x2APIC EOI MSR to the guest through the VMCS MSR bitmap. The guest writes directly to the physical APIC's EOI register without VM Exit, completing the zero-exit lifecycle for the entire interrupt.

### Section 3: Comparison Table

| Aspect | Traditional KVM | Sekiyama (Direct Delivery) |
|--------|----------------|---------------------------|
| Interrupt path | Device → pCPU → VM Exit → Host → Inject → Guest | Device → pCPU → **direct jump to Guest IDT** |
| VM Exit | Every interrupt | Almost never (only NMI or exceptions) |
| CPU utilization | Host/KVM consumes significant CPU on VM Exit | Nearly all CPU cycles go to the guest |
| Latency | High (microseconds, depends on VM Exit cost) | Very low (approaching bare-metal native interrupt latency) |
| EOI overhead | Yes (VM Exit) | None (Direct EOI) |

### Section 4: Advantages and Disadvantages

**Advantages:**
1. **Ultimate performance** — Near-bare-metal interrupt latency, ideal for network forwarding, high-frequency trading
2. **Hardware-independent** — Does not require APICv or Posted Interrupts CPU features
3. **CPU efficiency** — Host OS consumes almost zero cycles on the dedicated CPU

**Disadvantages and limitations:**
1. **Poor compatibility** — Requires modified KVM kernel code (`KVM_SET_SLAVE_CPU` is non-standard), difficult to merge upstream
2. **Resource waste** — Must sacrifice an entire physical CPU per vCPU; no overcommitment possible
3. **Debugging difficulty** — Since interrupts bypass the host, the host cannot monitor or intervene in the guest's internal interrupt state
4. **Configuration complexity** — Requires precise IRQ affinity configuration; if host interrupts are misrouted to the isolated CPU, they will be silently ignored, potentially causing **system hangs**

### Section 5: Modern Evolution

The article explicitly frames this approach as the **"software predecessor"** to APICv (Virtual Interrupt Delivery) and Posted Interrupts:
- APICv achieves similar zero-exit interrupt delivery through hardware, without sacrificing host CPU scheduling capability
- Posted Interrupts further optimize handling when a vCPU is preempted

The conclusion notes that while the approach has been superseded by hardware features, its design principles (CPU isolation + direct delivery + NMI fallback) remain essential for understanding virtualization interrupt architecture.

## Key Themes

- **Software vs. hardware approaches** — The article highlights how a software-only approach can achieve similar performance to later hardware features, at the cost of flexibility and maintainability
- **Explicit trade-off analysis** — Provides the clearest enumeration of disadvantages (debugging opacity, system hang risk from IRQ misconfiguration) not found in the original presentation slides
- **Historical positioning** — Frames the Sekiyama scheme as a stepping stone in the VM Exit elimination timeline, from pure-software workarounds to hardware-assisted solutions

## Relationship to Other Sources

- Directly analyzes the same work as [src-lcna-co2012-sekiyama](src-lcna-co2012-sekiyama.md) (the original LinuxCon 2012 slides) — this article reorganizes and adds commentary
- Complements [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md) — adds explicit disadvantage analysis to the Direct Interrupt Delivery section
- Relates to [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md) — reinforces the progressive VM Exit elimination narrative
- Contrast with APICv-based approaches in [analysis-vm-exit-reduction-and-timer-virtualization](../analyses/analysis-vm-exit-reduction-and-timer-virtualization.md)

## See also

- [src-lcna-co2012-sekiyama](src-lcna-co2012-sekiyama.md)
- [analysis-direct-interrupt-delivery-sekiyama](../analyses/analysis-direct-interrupt-delivery-sekiyama.md)
- [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md)
- [kvm-cpu-virtualization](../entities/kvm-cpu-virtualization.md)
- [kvm-performance-tuning](../entities/kvm-performance-tuning.md)
- [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md)
- [analysis-vm-exit-reduction-and-timer-virtualization](../analyses/analysis-vm-exit-reduction-and-timer-virtualization.md)
