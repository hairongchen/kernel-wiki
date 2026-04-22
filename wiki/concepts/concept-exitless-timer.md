---
type: concept
created: 2026-04-10
updated: 2026-04-10
sources: [all-solution-vmexit, bytedance-solution-vmexit, minimizing-vmexits-pv-ipi-passthrough-timer, lcna-co2012-sekiyama, kvm-devirt-kvmforum2022]
tags: [vm-exit, timer, exitless-timer, kvm, lapic, tsc-deadline, posted-interrupt, timer-passthrough, performance]
---

# Exit-less Timer

Exit-less timer refers to a family of techniques that eliminate or minimize VM Exits caused by guest timer operations in KVM. Timer-related VM Exits are typically the **dominant exit source** in CPU-isolated KVM environments, making exit-less timer critical for near-bare-metal virtualization performance.

## The Problem: Timer VM Exit Overhead

In standard KVM with Intel TSC-Deadline mode (the most common modern timer mode), a single guest timer event produces **at least two VM Exits**:

```
Guest:  write TSC_DEADLINE MSR ──── timer elapses ──── timer handler ──── EOI
              │                          │                                 │
         VM Exit #1                 VM Exit #2                       (VM Exit #3
         (MSR_WRITE)              (EXTERNAL_INTERRUPT)              with non-APICv)
              │                          │
Host:    KVM emulates              host receives
         LAPIC timer               timer interrupt,
         programming               injects vIRQ
```

Production measurements show:
- **93%** of external interrupt exits are caused by local timer (vector 236)
- **62.5%** of MSR_WRITE exits are caused by TSC_DEADLINE programming
- Together these two categories dominate the VM Exit profile

## Three Vendor Approaches

### 1. Tencent: Exitless Timer via Posted Interrupt (Upstream)

**How it works**: When the guest writes TSC_DEADLINE MSR, KVM intercepts (VM Exit still occurs) and programs the timer on a **different pCPU**. When the timer fires, that pCPU uses the Posted-Interrupt mechanism to inject the timer interrupt into the guest's pCPU without causing a VM Exit.

```
Guest pCPU:   write TSC_DEADLINE ──── [timer fires on other pCPU] ──── timer handler
                    │                        no VM Exit!
               VM Exit (MSR_WRITE)
                    │
Host:          program timer on       other pCPU: posted-interrupt
               other pCPU's LAPIC     inject to guest pCPU
```

**Trade-off**: Simple, upstream-merged (see `pi_inject_timer`). Still has the MSR_WRITE VM Exit for timer programming. Requires `nohz_full` CPU isolation.

**Patch**: `[v7] KVM: LAPIC: Implement Exitless Timer` — `kvm_set_lapic_tscdeadline_msr()` routes to `kvm_lapic_expired_hv_timer()` on a different pCPU.

### 2. Alibaba: PV Timer via Shared Page (RFC, not merged)

**How it works**: Guest and KVM share a memory page. The guest writes timer configuration to this shared page instead of the TSC_DEADLINE MSR, avoiding the MSR_WRITE VM Exit entirely. Other host pCPUs poll the shared pages to detect timer programming requests.

```
Guest pCPU:   write to shared page ──── [host pCPU polls, programs timer] ──── timer fires
                  no VM Exit!                                                  posted-interrupt
```

**Trade-off**: Eliminates MSR_WRITE exit, but wastes CPU cycles on continuous polling. The polling pCPUs cannot be used for other work. Not merged upstream.

**Patch**: `[RFC,0/7] kvm pvtimer` — introduces `MSR_KVM_PV_TIMER_EN` for shared-page timer.

### 3. ByteDance: Timer Passthrough (RFC, not merged)

**How it works**: The physical LAPIC Timer is **directly assigned to the guest**. The host is forced to use early Broadcast Timer mode, freeing each CPU's local APIC timer for exclusive guest use. The guest writes TSC_DEADLINE MSR directly to hardware (MSR interception disabled), and the timer interrupt is delivered directly to the guest via the interrupt non-exit mechanism.

```
Guest pCPU:   write TSC_DEADLINE ──── timer fires ──── timer handler
                 no VM Exit!          no VM Exit!       no VM Exit!
              (direct HW write)    (direct delivery)
```

**Trade-off**: Eliminates **all** timer VM Exits (both programming and firing). Most complex implementation — requires host timer mode changes, dynamic context switching of timer state on VM entry/exit, and coordination with the interrupt non-exit framework. Not merged upstream.

**Patch**: `[RFC: timer passthrough 0/9] Support timer passthrough for VM`

## Comparison

| Aspect | Tencent | Alibaba | ByteDance |
|--------|---------|---------|-----------|
| MSR_WRITE exit eliminated? | No | Yes | Yes |
| EXTERNAL_INTERRUPT exit eliminated? | Yes | Yes | Yes |
| Additional CPU cost | None | Polling pCPUs | Host broadcast timer |
| Guest modification needed? | No | Yes (PV) | No |
| Upstream status | Merged | RFC (not merged) | RFC (not merged) |
| Complexity | Low | Medium | High |
| Timer latency reduction | Partial | Partial | ~12.5% (approaching bare metal) |

## Prerequisites

All three approaches require:

1. **CPU isolation**: `isolcpus` to dedicate physical CPUs to vCPUs
2. **nohz_full**: Disable tick on isolated CPUs (static via kernel cmdline, or dynamic in ByteDance's approach)
3. **CPU pinning**: vCPU threads pinned 1:1 to physical CPUs
4. **APICv**: Hardware APIC virtualization must be enabled

## Relationship to Broader VM Exit Elimination

Exit-less timer is one component of a broader effort to achieve zero-VM-Exit virtualization:

- **Timer**: Exit-less timer (this page)
- **IPI**: PV IPI / IPI fastpath / Passthrough IPI (see [concept-hardware-virtualization](concept-hardware-virtualization.md))
- **Device interrupts**: VFIO interrupt passthrough, direct delivery
- **MSR access**: Selective MSR bitmap control
- **Preemption timer**: Eliminated via nohz_full + timer passthrough

The progression of hardware features (VT-x → EPT → APICv → Posted Interrupts → APIC Timer hardware acceleration) systematically reduces VM Exits, but software techniques like exit-less timer bridge the gap until hardware support is widely available.

ByteDance's [KVM-devirt](../entities/kvm-devirt.md) (KVM Forum 2022) integrates timer passthrough as one of six techniques in a comprehensive zero-overhead partition hypervisor. In that context, BM uses the physical LAPIC timer directly while the host falls back to broadcast timer, and KVM maps the TSC offset into the guest so it can program MSR_TSCDEADLINE with host-adjusted values. Timer latency benchmarks show BM matching native host on both Intel and AMD platforms.

## See also

- [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md)
- [kvm-performance-tuning](../entities/kvm-performance-tuning.md)
- [concept-hardware-virtualization](concept-hardware-virtualization.md)
- [analysis-vm-exit-reduction-and-timer-virtualization](../analyses/analysis-vm-exit-reduction-and-timer-virtualization.md)
- [src-all-solution-vmexit](../sources/src-all-solution-vmexit.md)
- [kvm-devirt](../entities/kvm-devirt.md)
- [src-kvm-devirt-kvmforum2022](../sources/src-kvm-devirt-kvmforum2022.md)
- [src-bytedance-solution-vmexit](../sources/src-bytedance-solution-vmexit.md)
