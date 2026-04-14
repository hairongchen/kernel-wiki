---
type: source
created: 2026-04-10
updated: 2026-04-10
title: "Minimizing VMExits in Private Cloud by Aggressive PV IPI and Passthrough Timer"
sources: [minimizing-vmexits-pv-ipi-passthrough-timer]
tags: [kvm, vm-exit, timer-passthrough, pv-ipi, posted-interrupt, lapic-timer, performance]
---

# Source: Minimizing VMExits in Private Cloud by Aggressive PV IPI and Passthrough Timer

**Authors:** Huaqiao & Yibo Zhou, ByteDance
**Date:** ~2020
**Format:** Presentation slides (20 slides)
**Coverage:** Two techniques to minimize VM Exits in large-scale private cloud deployments: timer passthrough (guest directly programs physical LAPIC timer) and NoExit PVIPI (guest sends IPIs via posted-interrupt hardware without VM Exit).

## Summary

This presentation addresses two major sources of VM Exits in ByteDance's private cloud: timer operations and inter-processor interrupts (IPIs). Large VMs (72-104 vCPUs) experience extremely high VM Exit rates (250K-550K timer/IPI exits per 5 minutes). The authors propose two solutions that eliminate these VM Exits by allowing the guest to directly access hardware timer and IPI mechanisms.

### Background: The Problems (Slides 3-5)

**Problem 1 — Timer Exits:**
- Timer arming, disarming, and firing all incur VM Exits
- High frequency of timer reprogramming (arm/disarm) in their workloads
- Each timer cycle involves multiple VM Exit/Entry pairs: set timer → VM Exit → emulate → VM Enter → ... → cancel timer → VM Exit → emulate → VM Enter → ... → timer fires → VM Exit → IRQ inject → VM Enter

**Problem 2 — IPI Exits:**
- Large VMs (T1: 72 vCPUs/376GB, T2: 104 vCPUs/187GB) generate high IPI traffic
- 250K-550K IPI-caused VM Exits per 5-minute window (measured in production)
- Not well addressed by existing PV IPI solutions

### Existing Solutions (Slides 7, 13)

**Exitless timer approaches:**
- **Wanpeng Li (Tencent Cloud):** Exitless timer via posted interrupt — requires housekeeping CPUs, injects expired timer interrupt via posted-interrupt mechanism (merged upstream as `pi_inject_timer`)
- **Yang Zhang (Alibaba Cloud):** PV timer — requires guest kernel modification for the PV feature, dedicated CPU must be reserved

**Exitless IPI approach:**
- **Wanpeng Li (Tencent Cloud):** Exitless IPI — marks all destination CPUs in a bitmap, sends IPIs to all CPUs via a single hypercall, VMM scans bitmap and sends IPIs one by one (merged upstream as KVM PV IPI)

### Solution 1: Timer Passthrough (Slides 8-12)

A new exitless timer approach where the VM directly accesses the physical LAPIC timer hardware:

**Core idea:**
- The VM accesses the physical LAPIC timer directly (no emulation, no VM Exit)
- Host timer is offloaded to the VMX preemption timer during VM execution
- Timer interrupt delivered to VM via external interrupt exit (not emulated vIRQ injection)

**Implementation details:**

1. **Physical LAPIC timer access:**
   - Guest LAPIC timer must operate in TSC-deadline mode
   - Disable intercept of TSC-deadline MSR (remove from MSR bitmap)
   - Guest writes directly to the physical `IA32_TSC_DEADLINE` MSR
   - TSC value adjustment on VM entry: `vm_tsc_value = host_tsc_value * TSC_multiplier + offset`

2. **Host timer offloading to preemption timer:**
   - On VM enter: find the nearest-expiring host timer, offload it to the VMX preemption timer
   - On VM exit / vCPU preblock: restore the host timer to the physical LAPIC timer, restore the VM timer to a software-emulated timer in the VMM
   - On preemption timer expiry: call the host clock event handler

3. **Timer interrupt delivery:**
   - When the physical LAPIC timer fires during guest execution, it causes an external interrupt exit (not a timer-specific emulation exit)
   - KVM injects the timer interrupt into the VM on re-entry

**Comparison: Normal vs. Passthrough timer flow:**
- Normal: set timer → VM Exit → emulate → VM Enter → ... → timer fires → VM Exit → IRQ inject → VM Enter (2+ VM Exits)
- Passthrough: set timer → direct physical access (no exit) → timer fires → external interrupt exit → IRQ inject → VM Enter (1 VM Exit only on fire, 0 on set/cancel)

**Evaluation (Intel Xeon Platinum 8260 @ 2.40GHz):**
- memcached: +35.5% ops/sec improvement (Sets: 352K → 478K, Gets: 70K → 95K)
- cyclictest average latency: 10μs → 7μs (30% reduction)

### Solution 2: NoExit PVIPI (Slides 14-18)

A zero-VM-Exit IPI mechanism that leverages the posted-interrupt hardware:

**Core idea:**
- Pass through the posted-interrupt descriptor (`pi_desc`) to the guest
- Disable interception of MSR.ICR (Interrupt Command Register)
- Guest sends IPIs directly by programming the posted-interrupt hardware — no VM Exit at all
- Provide `MSR_KVM_PV_ICR` as a fallback for special interrupt types (SMI, NMI, SIPI) that still need VMM mediation

**Guest-side IPI flow (vcpu0 → vcpu1):**
1. Get the `pi_desc` of the target vcpu1
2. `atomic_test_and_set` the PIR (Posted Interrupt Request) bit for the desired vector
3. `atomic_test_and_set` the ON (Outstanding Notification) bit
4. Get NV (Notification Vector) from `pi_desc.NV`
5. Prepare ICR value with the notification vector and destination
6. `wrmsr ICR` — the hardware sends the notification IPI, which the target CPU processes via the posted-interrupt mechanism

**Host-side setup:**
- Disable MSR.ICR intercept when NoExit PVIPI is enabled in guest
- Passthrough `pi_desc` addresses to the VM (guest can read the target vCPU's pi_desc)
- RFC patch: `https://patchwork.kernel.org/patch/11759063/`

**Evaluation (Intel Xeon Gold 5218 @ 2.30GHz):**
- Single IPI operation cost: bare-metal 228 cycles, VM 11,486 cycles, VM with NoExit PVIPI **412 cycles** (96.4% reduction from vanilla VM, only 1.8x bare-metal)
- memcached: +14.8% ops/sec improvement (Sets: 315K → 361K, Gets: 31K → 36K)

### Future Work (Slide 19)

- **NoExit PVIPI:** Security hardening via EPTP Switch feature by VMFUNC (prevent guest from accessing/corrupting pi_desc of other guests)
- **Passthrough Timer:** Support live migration (dynamically turn on/off the feature during migration)

## Key Themes

- **Production-driven optimization** — both techniques motivated by real VM Exit measurements in ByteDance's private cloud with large VMs (72-104 vCPUs)
- **Hardware mechanism reuse** — timer passthrough reuses the physical LAPIC timer + VMX preemption timer; NoExit PVIPI reuses the posted-interrupt hardware path
- **Complementary to upstream** — builds on Wanpeng Li's exitless timer (pi_inject_timer) and PV IPI but goes further by eliminating the remaining VM Exits
- **Security vs. performance trade-off** — NoExit PVIPI exposes pi_desc to guests, creating a potential attack surface that needs EPTP Switch hardening

## Relationship to Other Sources

- Directly extends [analysis-vm-exit-reduction-and-timer-virtualization](../analyses/analysis-vm-exit-reduction-and-timer-virtualization.md) — timer passthrough represents a more aggressive optimization beyond the posted timer interrupt approach already documented
- Builds on APICv/posted-interrupt infrastructure documented in [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md) — NoExit PVIPI leverages the same pi_desc mechanism but exposes it to the guest
- Related to Sekiyama's direct interrupt delivery ([src-lcna-co2012-sekiyama](src-lcna-co2012-sekiyama.md)) — both aim to eliminate interrupt-path VM Exits, but Sekiyama uses CPU isolation while ByteDance uses posted-interrupt hardware
- Extends [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md) — demonstrates the ongoing trajectory toward zero-VM-Exit virtualization

## See also

- [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md)
- [analysis-vm-exit-reduction-and-timer-virtualization](../analyses/analysis-vm-exit-reduction-and-timer-virtualization.md)
- [kvm-performance-tuning](../entities/kvm-performance-tuning.md)
- [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md)
- [src-lcna-co2012-sekiyama](src-lcna-co2012-sekiyama.md)
