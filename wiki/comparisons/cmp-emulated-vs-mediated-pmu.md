---
type: comparison
created: 2026-04-10
updated: 2026-04-10
sources: [qemu-kvm-source-code-and-application, sean-jc-mediated-pmu-branch]
tags: [kvm, pmu, performance-monitoring, virtualization, perf, vpmu]
---

# Emulated PMU vs Mediated PMU

KVM provides two architecturally distinct approaches for virtualizing the x86 Performance Monitoring Unit (PMU) to guest VMs: the traditional **Emulated PMU** (based on the host `perf` subsystem) and the newer **Mediated PMU** (direct hardware counter access). This page compares their designs, trade-offs, and implementation details based on the upstream KVM codebase and Sean Christopherson's mediated PMU patch series (`vmx/mediated_pmu_freeze_in_smm` branch, lore: `20260207041011.913471-*-seanjc@google.com`).

## Overview

| Dimension | Emulated PMU | Mediated PMU |
|-----------|-------------|--------------|
| Core mechanism | KVM programs host `perf_event` on behalf of the guest | KVM loads guest PMU state directly into hardware MSRs |
| Counter ownership | Host perf subsystem owns hardware counters | Guest "borrows" hardware counters during VM-Entry |
| MSR access | All PMU MSR accesses cause VM-Exits | Most PMU MSR accesses are passthrough (no VM-Exit) |
| PMI delivery | Via perf overflow callback → KVM injection | Via hardware PMI → `kvm_handle_guest_mediated_pmi()` |
| Accuracy | Approximate (sampling-based, skids) | Hardware-precise (counts run in real hardware) |
| Host-guest isolation | Strong (perf mediates all access) | Context-switch based (load/put at VM-Entry/Exit) |
| Hardware requirements | Any PMU version >= 1 | Intel PMU v4+, full-width writes, VMCS PERF_GLOBAL_CTRL load |
| Multiplexing with host | Automatic (perf schedules events) | Exclusive during guest execution (host perf disabled) |

## Architecture

### Emulated PMU: perf as the Backend

The emulated PMU treats the guest's PMU as a *virtual device*. When the guest programs a performance counter (writes to `IA32_PERFEVTSELx`, `IA32_PMCx`, etc.), every MSR access causes a VM-Exit. KVM intercepts these writes and translates them into host `perf_event` operations:

```
Guest writes MSR_P6_EVNTSEL0
  → VM-Exit (MSR write)
  → kvm_pmu_set_msr()
    → reprogram_counter()
      → pmc_pause_counter()           // pause existing perf_event
      → pmc_reprogram_counter()       // create new perf_event
        → perf_event_create_kernel_counter()
          attr.exclude_host = 1        // only count in guest context
          attr.sample_period = f(pmc->counter)
```

Key characteristics:

- **Every counter operation goes through perf**: Reading a counter (`RDPMC`, `RDMSR`) accumulates counts from `perf_event_pause()`. Writing a counter adjusts an offset relative to the perf event's current count.
- **Overflow detection is sampling-based**: perf generates an interrupt when the counter crosses zero (based on `sample_period`). The overflow callback `kvm_perf_overflow()` sets the corresponding bit in `global_status` and requests PMI injection into the guest.
- **Emulated instruction counting**: For instructions emulated by KVM (which perf cannot see), KVM maintains a separate `pmc->emulated_counter` that is folded into the total count during `pmc_pause_counter()`.

```
pmc_pause_counter():
  counter += perf_event_pause(perf_event, true)   // hardware counts
  counter += pmc->emulated_counter                 // software counts
  pmc->counter = counter & bitmask
  pmc->emulated_counter = 0
```

- **Event scheduling**: The host perf subsystem owns the physical counters. If the host also wants to use PMU counters, perf multiplexes between host and guest events. Cross-mapping detection (`intel_pmu_cross_mapped_check()`) handles cases where guest and host events compete for the same physical counter.

### Mediated PMU: Direct Hardware Access

The mediated PMU gives the guest *direct access* to the physical PMU hardware. Instead of translating guest PMU operations through perf, KVM loads the guest's raw PMU MSR values into hardware at VM-Entry and reads them back at VM-Exit:

```
VM-Entry path:
  kvm_mediated_pmu_load()
    → perf_load_guest_context()        // tell perf to step aside
    → wrmsrq(PERF_GLOBAL_CTRL, 0)     // disable all counters
    → perf_load_guest_lvtpc(APIC_LVTPC) // set PMI delivery vector
    → kvm_pmu_load_guest_pmcs()        // write guest counters & selectors to HW
    → intel_mediated_pmu_load()        // sync GLOBAL_STATUS, FIXED_CTR_CTRL
    → vmcs_write64(GUEST_IA32_PERF_GLOBAL_CTRL, global_ctrl)
                                       // atomically enable at VM-Entry

VM-Exit path:
  kvm_mediated_pmu_put()
    → intel_mediated_pmu_put()         // read back GLOBAL_STATUS, clear HW state
    → kvm_pmu_put_guest_pmcs()         // rdpmc() all counters back to pmc->counter
    → perf_put_guest_lvtpc()           // restore host PMI vector
    → perf_put_guest_context()         // return control to host perf
```

Key characteristics:

- **MSR passthrough**: When the guest has the same number of counters as the host hardware, PMU MSR accesses do NOT cause VM-Exits. The guest reads/writes counters and event selectors at native speed. This is controlled by the MSR bitmap in `vmx_recalc_pmu_msr_intercepts()`.
- **Counter values are physical**: `pmc->counter` directly reflects the hardware counter value. Writes are simple: `pmc->counter = val & pmc_bitmask(pmc)`. No perf event, no offset arithmetic, no emulated counter.
- **PERF_GLOBAL_CTRL via VMCS**: On Intel, `PERF_GLOBAL_CTRL` is loaded/saved atomically via dedicated VMCS fields (`VM_ENTRY_LOAD_IA32_PERF_GLOBAL_CTRL`, `VM_EXIT_SAVE_IA32_PERF_GLOBAL_CTRL`). This eliminates any window where guest event selectors are active with host global control or vice versa.
- **PMI is hardware-native**: When a mediated counter overflows, the hardware generates a real PMI. The NMI/PMI handler calls `kvm_handle_guest_mediated_pmi()`, which simply sets `KVM_REQ_PMI` on the vCPU. No perf event callback involved.
- **Emulated instruction counting is direct**: For emulated instructions, the mediated path simply increments `pmc->counter` and checks for wrap-around. No `emulated_counter` accumulator needed.

```
kvm_pmu_incr_counter() [mediated path]:
  pmc->counter = (pmc->counter + 1) & pmc_bitmask(pmc)
  if (!pmc->counter) {
      pmu->global_status |= BIT_ULL(pmc->idx)
      if (pmc_is_pmi_enabled(pmc))
          kvm_make_request(KVM_REQ_PMI, vcpu)
  }
```

## Detailed Comparison

### 1. VM-Exit Overhead

**Emulated PMU**: Every access to a PMU MSR causes a VM-Exit. For a workload that frequently reads/writes performance counters (e.g., `perf stat` inside a guest, or a profiling daemon), this creates significant overhead. Each VM-Exit costs hundreds to thousands of cycles for state save/restore alone, plus the kernel-side handling.

**Mediated PMU**: PMU MSR accesses are passthrough when the guest has the full set of hardware counters. `RDPMC`, `RDMSR`/`WRMSR` to `PERFCTRx`, `PERFEVTSELx`, and `FIXED_CTRx` all execute at native speed. Intercepts are only needed when:
- The guest has fewer counters than hardware (to prevent access to non-existent counters)
- `PERF_GLOBAL_CTRL` needs interception (counter subset scenarios)

This is configured in `vmx_recalc_pmu_msr_intercepts()`:

```c
bool intercept = !has_mediated_pmu;
// For each GP counter in guest range: set intercept = !has_mediated_pmu
// For counters beyond guest range: always intercept
// For GLOBAL_STATUS/CTRL/OVF_CTRL: intercept if counter subset
```

### 2. Counter Accuracy and Precision

**Emulated PMU**: Counts are *approximate*. The perf subsystem uses sampling: it programs the hardware counter with a negative period value and waits for overflow. Between the point of overflow and when KVM processes the event, additional counts accumulate ("skid"). Furthermore:
- The `emulated_counter` path for KVM-emulated instructions adds to imprecision
- Counter reads require `perf_event_pause()` to get an accurate snapshot
- Context switches between host and guest perf events can lose or double-count events at boundaries

**Mediated PMU**: Counts are *hardware-precise*. The guest's counters run in actual hardware with the guest's exact event selector configuration. Counter values read back via `rdpmc()` at VM-Exit reflect the true hardware count. The only approximation is for KVM-emulated instructions, which increment counters in software.

### 3. Host-Guest PMU Coexistence

**Emulated PMU**: Host and guest PMU usage coexist naturally through perf's event scheduling. The `exclude_host = 1` attribute ensures guest events only count during guest execution. Multiple guests and the host can all share the PMU hardware through time-multiplexing managed by perf. However, this flexibility comes at the cost of cross-mapping issues and event scheduling overhead.

**Mediated PMU**: The guest has **exclusive PMU access** during its execution. `perf_load_guest_context()` tells the host perf subsystem to step aside entirely. While the guest is running, host perf events cannot use the PMU. This means:
- No host profiling of guest code via `perf_event` while mediated PMU is active
- The host recovers the PMU at VM-Exit via `perf_put_guest_context()`
- The trade-off is simplicity and performance for exclusivity

### 4. Event Filter Enforcement

**Emulated PMU**: Event filtering is enforced by *not programming* the perf event when the filter denies an event. `reprogram_counter()` checks `pmc_is_event_allowed()` and skips creating the perf event if the event is denied.

**Mediated PMU**: Event filtering works differently since counter MSR accesses are passthrough. Filtering is enforced by manipulating the **hardware-visible enable bits**. For GP counters, the `ENABLE` bit in the event selector is masked in `eventsel_hw`. For fixed counters, the corresponding bits in `fixed_ctr_ctrl_hw` are masked. The guest sees its own values, but the hardware runs with the filtered version:

```c
kvm_mediated_pmu_refresh_event_filter():
  if (pmc_is_gp(pmc)) {
      pmc->eventsel_hw &= ~ENABLE;
      if (allowed)
          pmc->eventsel_hw |= pmc->eventsel & ENABLE;
  } else {
      pmu->fixed_ctr_ctrl_hw &= ~mask;
      if (allowed)
          pmu->fixed_ctr_ctrl_hw |= pmu->fixed_ctr_ctrl & mask;
  }
```

### 5. Hardware Requirements

**Emulated PMU**: Works with any Intel PMU version >= 1 (or AMD equivalent). Minimal hardware requirements since perf abstracts the hardware details.

**Mediated PMU**: Requires specific hardware features (`intel_pmu_is_mediated_pmu_supported()`):
- **PMU Architectural Version >= 4**: Needed for `MSR_CORE_PERF_GLOBAL_STATUS_SET`, which allows KVM to precisely load the guest's overflow status bits
- **Full-Width Writes** (`PERF_CAP_FW_WRITES`): Required for `MSR_IA32_PMCx` (as opposed to `MSR_IA32_PERFCTRx`) to write full 48-bit counter values without sign-extension issues
- **VMCS PERF_GLOBAL_CTRL fields**: `VM_ENTRY_LOAD_IA32_PERF_GLOBAL_CTRL` must be supported for atomic enable/disable at VM transitions

### 6. Context Switch Cost

**Emulated PMU**: Minimal context switch overhead at VM-Entry/Exit because perf events are persistent host objects. The events count only during guest execution (`exclude_host = 1`). However, cross-mapping checks and perf event scheduling add latency.

**Mediated PMU**: Context switches require explicit save/restore of all PMU MSRs. At `kvm_mediated_pmu_load()`: write N GP counters + N GP selectors + N fixed counters + FIXED_CTR_CTRL + GLOBAL_STATUS. At `kvm_mediated_pmu_put()`: read back N GP counters + N fixed counters + GLOBAL_STATUS + clear hardware state. For 8 GP + 4 fixed counters, this is ~30 MSR reads/writes per VM-Entry/Exit cycle. This fixed cost is paid regardless of whether the guest is actively using the PMU.

### 7. Nested Virtualization

**Emulated PMU**: Works naturally with nested virtualization since the PMU is fully software-emulated through perf.

**Mediated PMU**: Interactions with nested virtualization (L2 guest running inside L1 hypervisor) require careful handling. The `vmx/mediated_pmu_freeze_in_smm` branch name suggests ongoing work on edge cases like SMM (System Management Mode) interactions where PMU state must be frozen.

## When to Use Which

| Scenario | Recommendation |
|----------|---------------|
| Production VM profiling with minimal overhead | Mediated PMU |
| Host-side profiling of guest workloads | Emulated PMU |
| Mixed host + guest PMU usage | Emulated PMU |
| Guest running `perf` tools for self-profiling | Mediated PMU |
| Hardware without PMU v4 / full-width writes | Emulated PMU (only option) |
| Maximum counter accuracy inside guest | Mediated PMU |
| Counter subset (fewer guest than host counters) | Either (mediated adds intercepts for missing counters) |

## Implementation Summary

```
                    Emulated PMU                          Mediated PMU
                    ============                          =============

MSR Write:     VM-Exit → KVM trap                   Direct HW write (no exit)
               → reprogram_counter()
               → perf_event_create_kernel_counter()

Counter Read:  VM-Exit → KVM trap                   Direct RDPMC (no exit)
               → perf_event_pause() + emulated_counter

Counting:      perf_event in host context            Hardware counter in guest context
               (exclude_host=1)                      (loaded via MSR writes at VM-Entry)

Overflow:      perf callback → kvm_perf_overflow()   Hardware PMI → kvm_handle_guest_mediated_pmi()
               → KVM_REQ_PMI                         → KVM_REQ_PMI

Enable/Dis:    perf_event_enable/disable              VMCS GUEST_IA32_PERF_GLOBAL_CTRL
                                                      (atomic at VM-Entry/Exit)

Host coexist:  Multiplexed via perf scheduler        Exclusive (perf_load/put_guest_context)

Key code:      pmu.c: reprogram_counter()            pmu.c: kvm_mediated_pmu_load/put()
               pmu.c: pmc_reprogram_counter()        pmu_intel.c: intel_mediated_pmu_load/put()
               pmu.c: kvm_perf_overflow()            vmx.c: vmx_recalc_pmu_msr_intercepts()
```

## Relationship to Hardware Virtualization Trajectory

The mediated PMU follows the same pattern as other hardware virtualization optimizations (see [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md)):

```
EPT      → eliminate memory VM-Exits         (shadow PT → hardware walk)
APICv    → eliminate interrupt VM-Exits       (software inject → posted interrupts)
VT-d     → eliminate I/O VM-Exits            (device emulation → passthrough)
Med. PMU → eliminate PMU VM-Exits            (perf emulation → direct HW access)
```

Each generation removes the hypervisor from another data path, approaching bare-metal performance. The mediated PMU is the natural extension of this trajectory to the performance monitoring subsystem.

## See also

- [kvm-pmu-virtualization](../entities/kvm-pmu-virtualization.md) -- KVM PMU virtualization data structures, MSR handling, key functions
- [kvm-cpu-virtualization](../entities/kvm-cpu-virtualization.md) -- VCPU lifecycle and VM-Entry/Exit mechanics
- [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md) -- PMI delivery and APIC emulation
- [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md) -- Progressive VM-Exit elimination
- [kvm-performance-tuning](../entities/kvm-performance-tuning.md) -- General KVM performance optimization
- [cmp-emulated-vs-mediated-pmu-zh](cmp-emulated-vs-mediated-pmu-zh.md) -- Chinese version / 中文版
