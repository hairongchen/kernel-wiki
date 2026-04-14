---
type: source
created: 2026-04-10
updated: 2026-04-10
sources: [all-solution-vmexit]
tags: [vm-exit, timer, exitless-timer, kvm, tencent, alibaba, bytedance, lapic, tsc-deadline, posted-interrupt]
---

# Source: KVM Timer 导致 Exit 过多的解决办法

| Field | Value |
|-------|-------|
| Author | @惠伟 (Hui Wei) |
| Platform | 知乎 (Zhihu) |
| URL | zhuanlan.zhihu.com/p/384786685 |
| Coverage | KVM timer VM Exit root-cause analysis + three vendor solutions |
| Kernel context | Linux 4.18+ (various vendor kernels) |

## Summary

A practitioner's investigation into excessive KVM VM Exits caused by timer operations, diagnosing the root causes via `kvm_stat` and kernel tracing, then surveying three Chinese cloud vendors' solutions (Tencent, Alibaba, ByteDance) for exit-less timer.

## Problem Analysis

### kvm_stat Diagnosis

Using `kvm_stat` on a production environment, the author identified three dominant VM Exit categories:

| Exit type | Count (vCPU 0) | Count (vCPU 1) |
|-----------|----------------|----------------|
| `MSR_WRITE` | 95,684 | 92,457 |
| `EXTERNAL_INTERRUPT` | 35,812 | 33,367 |
| `EPT_MISCONFIG` | 12,476 | 12,281 |

### Root Cause: External Interrupt

The VM-Exit interruption info field (`exit info` field with value `0x800000ec`) reveals vector 236 (`0xec`), which is the **local timer** interrupt. The physical machine uses CPU isolation (`isolcpus`), and the vCPU thread (PID 759956) is pinned to the isolated CPU. Local timer interrupts on this CPU cause external interrupt VM Exits.

Tracing confirms **93% of external interrupt exits** are caused by local timer (vector=236).

### Root Cause: MSR Write

MSR write tracing shows **62.5% of MSR_WRITE exits** are caused by writing `MSR_IA32_TSC_DEADLINE`, which is how the guest configures the LAPIC timer in TSC-Deadline mode.

### Timer VM Exit Mechanism

In virtualized environments, the hardware timer does not exist physically — it is emulated by KVM. When the guest writes the hardware timer (TSC_DEADLINE MSR), this causes a VM Exit for KVM to emulate. KVM can either use software timers or program the real hardware timer. When the hardware timer fires, the CPU receives an external interrupt, exits from guest mode, handles the interrupt, then injects the virtual timer interrupt back into the guest. This produces **at least two VM Exits per timer event** (one for MSR write, one for external interrupt).

## Solutions

### 1. Tencent: Exit-less Timer (Upstream)

**Approach**: When the guest writes the TSC_DEADLINE MSR, KVM configures the timer on a **different pCPU's local timer** instead of the local CPU. When the timer fires, the other pCPU uses posted-interrupt to inject the interrupt into the guest CPU without causing a VM Exit on the guest's pCPU.

**Trade-off**: The MSR_WRITE exit for configuring the timer still occurs — only the external interrupt exit is eliminated.

**Patch series**: `[v7] KVM: LAPIC: Implement Exitless Timer` (patchwork.kernel.org)
- `[v7,1/2] KVM: LAPIC: Make lapic timer unpinned`
- `[v7,2/2] KVM: LAPIC: Inject timer interrupt via posted interrupt`

**Status**: Merged upstream. Requires `nohz_full` isolation for the pCPU running the vCPU. See `pi_inject_timer`.

### 2. Alibaba: Exit-less Timer (kvm pvtimer, RFC)

**Approach**: Guest and KVM share a page. When the guest configures the timer, it writes to this **shared page** instead of writing the MSR, so there is no WRMSR exit. KVM uses other pCPUs to poll these shared pages and programs the timer accordingly.

**Trade-off**: Eliminates the MSR_WRITE exit, but requires other pCPUs to continuously poll the shared pages, which wastes CPU resources.

**Patch series**: `[RFC,0/7] kvm pvtimer` (lore.kernel.org)
- `[RFC,1/7] kvm: x86: emulate MSR_KVM_PV_TIMER_EN MSR`
- `[RFC,2/7] kvm: x86: add a function to exchange value`
- `[RFC,3/7] KVM: timer: synchronize tsc-deadline timestamp for guest`
- `[RFC,4/7] KVM: timer: program timer to a dedicated CPU`
- `[RFC,5/7] KVM: timer: ignore timer migration if pvtimer is enabled`
- `[RFC,6/7] Doc/KVM: introduce a new cpuid bit for kvm pvtimer`
- `[RFC,7/7] kvm: guest: reprogram guest timer`

**QEMU patch**: spinics.net/lists/kvm/m — not merged upstream.

### 3. ByteDance: Timer Passthrough + Exit-less Timer

**Approach**: Direct timer passthrough — the guest accesses the physical LAPIC timer directly. Compared to the Alibaba approach, this eliminates the CPU polling overhead. Compared to Tencent's approach, this also eliminates the WRMSR exit.

**Trade-off**: More complex implementation with more invasive host kernel changes (context switching between host and guest timer states).

**Patch series**: `[RFC: timer passthrough 0/9] Support timer passthrough for VM` (linux-kernel mailing list, 2021-02-06)
- 9-part series covering vmx hooks, host LAPIC timer offloading, preemption timer fallback, TSC offset passthrough, dynamic enable/close, state query

**Related talk**: KVM Forum "Minimizing VMExits in Private Cloud by Aggressive PV IPI and Passthrough Timer" by Huaqiao & Yibo Zhou.

## Key Themes

1. **Timer is the dominant VM Exit source** in CPU-isolated KVM environments — external interrupt (93% from local timer) and MSR_WRITE (62.5% from TSC_DEADLINE) together dominate the exit profile
2. **TSC-Deadline mode doubles the exit cost** — one exit for programming (MSR write) + one for firing (external interrupt)
3. **Three approaches trade off differently**: Tencent eliminates fire-exit but keeps program-exit (simplest, upstream); Alibaba eliminates program-exit but wastes CPU on polling; ByteDance eliminates both but has highest complexity
4. **All solutions require CPU isolation** as a prerequisite (isolcpus + nohz_full)

## Relationship to Other Sources

- Extends [src-minimizing-vmexits-pv-ipi-passthrough-timer](src-minimizing-vmexits-pv-ipi-passthrough-timer.md) by placing ByteDance's work in context with competing approaches
- Extends [src-lcna-co2012-sekiyama](src-lcna-co2012-sekiyama.md) by showing production-scale timer exit costs that motivated these solutions
- Directly related to [analysis-vm-exit-reduction-and-timer-virtualization](../analyses/analysis-vm-exit-reduction-and-timer-virtualization.md)

## See also

- [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md)
- [kvm-performance-tuning](../entities/kvm-performance-tuning.md)
- [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md)
