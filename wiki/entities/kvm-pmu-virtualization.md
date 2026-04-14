---
type: entity
created: 2026-04-10
updated: 2026-04-10
sources: [qemu-kvm-source-code-and-application, sean-jc-mediated-pmu-branch]
tags: [kvm, pmu, performance-monitoring, perf, vpmu, msr, counters]
---

# KVM PMU Virtualization

The KVM virtual Performance Monitoring Unit (vPMU) allows guest VMs to use hardware performance counters for profiling and monitoring. KVM supports two distinct PMU virtualization modes: the traditional **emulated PMU** (perf-based) and the newer **mediated PMU** (direct hardware access). This page covers the data structures, MSR handling, key functions, and operational flows common to both modes. For a side-by-side comparison, see [cmp-emulated-vs-mediated-pmu](../comparisons/cmp-emulated-vs-mediated-pmu.md).

## x86 PMU Architecture Background

The Intel Architectural Performance Monitoring facility (introduced in CPUID leaf `0AH`) provides:

- **General-purpose (GP) counters** (`IA32_PMCx` / `IA32_PERFCTRx`): programmable counters that count events specified by an associated event selector register (`IA32_PERFEVTSELx`).
- **Fixed counters** (`IA32_FIXED_CTR0..2`): count pre-defined events (instructions retired, core cycles, reference cycles). Controlled by `IA32_FIXED_CTR_CTRL`.
- **Global control registers**:
  - `IA32_PERF_GLOBAL_CTRL` -- enables/disables individual counters (both GP and fixed)
  - `IA32_PERF_GLOBAL_STATUS` -- overflow status bits for all counters
  - `IA32_PERF_GLOBAL_OVF_CTRL` (a.k.a. `GLOBAL_STATUS_RESET`) -- clears overflow bits
  - `IA32_PERF_GLOBAL_STATUS_SET` (v4+) -- sets overflow bits
- **PMU version** (from `CPUID.0AH:EAX[7:0]`): determines available features. KVM exposes up to version 2 to guests.

The PMU generates a **Performance Monitoring Interrupt (PMI)** when a counter overflows, delivered via the Local APIC's `LVTPC` (LVT Performance Counter) register.

## Core Data Structures

### struct kvm_pmc (Per-Counter State)

Defined in `arch/x86/include/asm/kvm_host.h`. Represents a single virtual performance counter:

```c
struct kvm_pmc {
    enum pmc_type type;          // KVM_PMC_GP or KVM_PMC_FIXED
    u8 idx;                      // Global PMC index (GP: 0..N-1, fixed: 32..34)
    bool is_paused;              // Whether the perf_event is paused [emulated only]
    bool intr;                   // PMI enabled for this counter

    u64 counter;                 // Counter value
                                 //   Emulated: base value relative to perf_event
                                 //   Mediated: actual hardware counter value

    u64 emulated_counter;        // Software-emulated event count [emulated only]
    u64 eventsel;                // Guest's event selector (IA32_PERFEVTSELx)
    u64 eventsel_hw;             // Hardware event selector [mediated: may mask ENABLE]
    struct perf_event *perf_event; // Host perf event [emulated only, NULL for mediated]
    struct kvm_vcpu *vcpu;       // Owning vCPU
    u64 current_config;          // Config of current perf_event [emulated only]
};
```

The `idx` field uses a global indexing scheme: GP counters occupy bits 0..31 and fixed counters occupy bits 32..63. This mirrors Intel's `PERF_GLOBAL_CTRL` bit layout.

### struct kvm_pmu (Per-vCPU PMU State)

Embedded in `struct kvm_vcpu_arch` as `vcpu->arch.pmu`:

```c
struct kvm_pmu {
    u8 version;                     // Architectural PMU version (max 2)
    unsigned nr_arch_gp_counters;   // Number of GP counters
    unsigned nr_arch_fixed_counters; // Number of fixed counters

    u64 fixed_ctr_ctrl;             // Guest's IA32_FIXED_CTR_CTRL
    u64 fixed_ctr_ctrl_hw;          // Hardware value (mediated: may mask bits)
    u64 global_ctrl;                // Guest's IA32_PERF_GLOBAL_CTRL
    u64 global_status;              // Guest's IA32_PERF_GLOBAL_STATUS
    u64 counter_bitmask[2];         // Max counter value masks [GP, fixed]
    u64 raw_event_mask;             // Valid event selector bits
    u64 reserved_bits;              // Reserved event selector bits

    struct kvm_pmc gp_counters[KVM_MAX_NR_GP_COUNTERS];       // max 8 (Intel)
    struct kvm_pmc fixed_counters[KVM_MAX_NR_FIXED_COUNTERS];  // max 3 (Intel)

    DECLARE_BITMAP(reprogram_pmi, X86_PMC_IDX_MAX);  // Counters needing reprogram
    DECLARE_BITMAP(all_valid_pmc_idx, X86_PMC_IDX_MAX);
    DECLARE_BITMAP(pmc_in_use, X86_PMC_IDX_MAX);
    DECLARE_BITMAP(pmc_counting_instructions, X86_PMC_IDX_MAX);
    DECLARE_BITMAP(pmc_counting_branches, X86_PMC_IDX_MAX);

    u64 pebs_enable;                // IA32_PEBS_ENABLE
    u64 ds_area;                    // IA32_DS_AREA
    u64 host_cross_mapped_mask;     // Cross-mapped counter mask [emulated only]
    bool need_cleanup;              // Perf event cleanup needed
    u8 event_count;                 // Number of active perf_events
};
```

### struct kvm_pmu_ops (Vendor Operations Table)

Defined in `arch/x86/kvm/pmu.h`. Dispatched via static calls for Intel vs AMD:

```c
struct kvm_pmu_ops {
    // Counter lookup
    struct kvm_pmc *(*rdpmc_ecx_to_pmc)(...);
    struct kvm_pmc *(*msr_idx_to_pmc)(...);
    int (*check_rdpmc_early)(...);

    // MSR handling
    bool (*is_valid_msr)(...);
    int (*get_msr)(...);
    int (*set_msr)(...);

    // Lifecycle
    void (*refresh)(...);
    void (*init)(...);
    void (*reset)(...);
    void (*deliver_pmi)(...);
    void (*cleanup)(...);

    // Mediated PMU operations
    bool (*is_mediated_pmu_supported)(struct x86_pmu_capability *host_pmu);
    void (*mediated_load)(struct kvm_vcpu *vcpu);
    void (*mediated_put)(struct kvm_vcpu *vcpu);
    void (*write_global_ctrl)(u64 global_ctrl);

    // Vendor-specific constants
    const u64 EVENTSEL_EVENT;
    const int MAX_NR_GP_COUNTERS;
    const int MIN_NR_GP_COUNTERS;
    const u32 PERF_GLOBAL_CTRL;
    const u32 GP_EVENTSEL_BASE;    // MSR_P6_EVNTSEL0 (Intel)
    const u32 GP_COUNTER_BASE;     // MSR_IA32_PMC0 (Intel)
    const u32 FIXED_COUNTER_BASE;  // MSR_CORE_PERF_FIXED_CTR0
    const u32 MSR_STRIDE;
};
```

Intel's implementation is `intel_pmu_ops` in `arch/x86/kvm/vmx/pmu_intel.c`. AMD's is `amd_pmu_ops` in `arch/x86/kvm/svm/pmu.c`.

## PMU Capability Detection and Initialization

### kvm_init_pmu_capability()

Called during KVM module initialization. Queries host PMU via `perf_get_x86_pmu_capability()` and establishes KVM's PMU capabilities:

```
kvm_init_pmu_capability():
  1. Disable vPMU for hybrid CPUs (X86_FEATURE_HYBRID_CPU)
  2. Query host PMU → kvm_host_pmu
  3. Validate minimum GP counter count
  4. Check mediated PMU support:
     - enable_pmu && enable_mediated_pmu && kvm_host_pmu.mediated
     - pmu_ops->is_mediated_pmu_supported(&kvm_host_pmu)
  5. Cap kvm_pmu_cap: version ≤ 2, counters ≤ MAX_NR
  6. Resolve event selectors for INSTRUCTIONS_RETIRED, BRANCH_INSTRUCTIONS_RETIRED
```

For Intel mediated PMU, `intel_pmu_is_mediated_pmu_supported()` requires:
- PMU version >= 4 (for `GLOBAL_STATUS_SET`)
- `PERF_CAP_FW_WRITES` (full-width counter writes)
- VMCS `LOAD_PERF_GLOBAL_CTRL` support

### Per-vCPU Initialization

```
kvm_pmu_init(vcpu):     // Allocate and zero PMU state
kvm_pmu_refresh(vcpu):  // Called on CPUID update
  → Set version, counter counts from guest CPUID
  → Compute bitmasks, reserved bits
  → Set initial global_ctrl (all GP counters enabled if version > 1)
  → For mediated: write_global_ctrl(global_ctrl)
  → Build all_valid_pmc_idx bitmap
```

## MSR Handling

### Guest MSR Reads and Writes

`kvm_pmu_get_msr()` and `kvm_pmu_set_msr()` handle all PMU-related MSR accesses. In emulated mode, every MSR access triggers a VM-Exit. In mediated mode, most counter/selector MSR accesses are passthrough (no VM-Exit) and these functions are only called for intercepted MSRs.

Key MSR handling in `kvm_pmu_set_msr()`:

| MSR | Handling |
|-----|----------|
| `PERF_GLOBAL_CTRL` | Update `pmu->global_ctrl`, reprogram changed counters. Mediated: also `write_global_ctrl()` to update VMCS |
| `PERF_GLOBAL_OVF_CTRL` | Clear bits in `pmu->global_status` |
| `PERF_GLOBAL_STATUS` | Read-only (write returns #GP) |
| `PERFEVTSELx` | Update `pmc->eventsel`, request counter reprogram |
| `PERFCTRx` / `PMCx` | `pmc_write_counter()` -- path diverges by mode |
| `FIXED_CTR_CTRL` | Update `pmu->fixed_ctr_ctrl`, reprogram affected fixed counters |

### Counter Write Path Divergence

```c
pmc_write_counter(pmc, val):
  if (mediated) {
      pmc->counter = val & pmc_bitmask(pmc);     // Direct store
      return;
  }
  // Emulated: offset relative to perf_event
  pmc->emulated_counter = 0;
  pmc->counter += val - pmc_read_counter(pmc);
  pmc->counter &= pmc_bitmask(pmc);
  pmc_update_sample_period(pmc);
```

### Counter Read Path Divergence

```c
pmc_read_counter(pmc):                            // in pmu.h
  if (mediated)
      return pmc->counter & pmc_bitmask(pmc);     // Direct read
  // Emulated: accumulate from perf + software
  counter = pmc->counter + pmc->emulated_counter;
  if (pmc->perf_event && !pmc->is_paused)
      counter += perf_event_read_value(perf_event, ...);
  return counter & pmc_bitmask(pmc);
```

## MSR Intercept Configuration (Mediated Mode)

`vmx_recalc_pmu_msr_intercepts()` configures which PMU MSRs trigger VM-Exits:

```
When mediated PMU is active:
  ┌─────────────────────┬──────────────────────────────────────────┐
  │ MSR                 │ Intercept?                               │
  ├─────────────────────┼──────────────────────────────────────────┤
  │ PERFCTRx (in range) │ No (passthrough)                         │
  │ PMCx (in range)     │ No (if fw_writes enabled)                │
  │ PERFEVTSELx         │ No (passthrough)                         │
  │ FIXED_CTRx (range)  │ No (passthrough)                         │
  │ PERFCTRx (excess)   │ Yes (guest has fewer than hardware)      │
  │ GLOBAL_STATUS       │ Only if counter subset                   │
  │ GLOBAL_CTRL         │ Only if counter subset                   │
  │ GLOBAL_OVF_CTRL     │ Only if counter subset                   │
  └─────────────────────┴──────────────────────────────────────────┘

VMCS controls:
  - VM_ENTRY_LOAD_IA32_PERF_GLOBAL_CTRL = enabled
  - VM_EXIT_SAVE_IA32_PERF_GLOBAL_CTRL = enabled (or autostore MSR)
  - HOST_IA32_PERF_GLOBAL_CTRL = 0 (disable host counters)
```

## Counter Reprogramming

### Emulated Path: reprogram_counter()

Called when a counter's event selector or enable state changes:

```
reprogram_counter(pmc):
  1. pmc_pause_counter()         // Pause perf_event, accumulate counts
  2. Check globally + locally enabled + event allowed
  3. If overflow detected → __kvm_perf_overflow()
  4. If config unchanged and resumable → pmc_resume_counter()
  5. Otherwise: pmc_release_perf_event() + pmc_reprogram_counter()
     → perf_event_create_kernel_counter()
       attr.type = PERF_TYPE_RAW
       attr.exclude_host = 1
       attr.sample_period = (-counter) & bitmask
```

### Mediated Path: kvm_mediated_pmu_refresh_event_filter()

Counter reprogramming in mediated mode is much simpler -- the hardware runs the guest's selectors directly. The only "reprogramming" is applying event filters:

```
kvm_mediated_pmu_refresh_event_filter(pmc):
  allowed = pmc_is_event_allowed(pmc)
  if GP counter:
      eventsel_hw: clear ENABLE, re-set if allowed
  if fixed counter:
      fixed_ctr_ctrl_hw: clear mask, re-set if allowed
```

## Emulated Instruction Counting

When KVM emulates an instruction, it must account for that instruction in active performance counters. The path diverges:

**Emulated mode** (`kvm_pmu_incr_counter` → emulated path):
```c
pmc->emulated_counter++;
kvm_pmu_request_counter_reprogram(pmc);
// Overflow detected later in reprogram_counter() → pmc_pause_counter()
```

**Mediated mode** (`kvm_pmu_incr_counter` → mediated path):
```c
pmc->counter = (pmc->counter + 1) & pmc_bitmask(pmc);
if (!pmc->counter) {                    // Wrapped to zero = overflow
    pmu->global_status |= BIT_ULL(pmc->idx);
    if (pmc_is_pmi_enabled(pmc))
        kvm_make_request(KVM_REQ_PMI, vcpu);
}
```

The mediated path is simpler because `pmc->counter` is the actual value (not an offset), so overflow detection is a straightforward zero check.

## Mediated PMU Context Switch

### VM-Entry: kvm_mediated_pmu_load()

```
kvm_mediated_pmu_load(vcpu):
  Guard: kvm_vcpu_has_mediated_pmu() && lapic_in_kernel()
  Assert: IRQs disabled

  1. perf_load_guest_context()
     // Tell host perf to stop using PMU hardware

  2. wrmsrq(PERF_GLOBAL_CTRL, 0)
     // Globally disable all counters before loading guest state

  3. perf_load_guest_lvtpc(APIC_LVTPC)
     // Set PMI delivery to use guest's LVTPC vector

  4. kvm_pmu_load_guest_pmcs(vcpu)
     // For each GP counter: write eventsel_hw, then counter to HW
     // For each fixed counter: write counter to HW

  5. intel_mediated_pmu_load(vcpu)      [vendor-specific]
     // Sync GLOBAL_STATUS: read HW, XOR with guest, set/clear delta
     // Write FIXED_CTR_CTRL to HW

  6. vmcs_write64(GUEST_IA32_PERF_GLOBAL_CTRL, global_ctrl)
     // VMCS atomically loads this at VM-Entry → counters start
```

### VM-Exit: kvm_mediated_pmu_put()

```
kvm_mediated_pmu_put(vcpu):
  Guard: kvm_vcpu_has_mediated_pmu() && lapic_in_kernel()
  Assert: IRQs disabled

  1. intel_mediated_pmu_put(vcpu)       [vendor-specific]
     // GLOBAL_CTRL already saved by VM-Exit hardware
     // Read GLOBAL_STATUS from HW → pmu->global_status
     // Clear HW GLOBAL_STATUS if non-zero
     // Clear HW FIXED_CTR_CTRL

  2. kvm_pmu_put_guest_pmcs(vcpu)
     // For each GP counter: rdpmc(i) → pmc->counter, clear HW
     // For each fixed counter: rdpmc() → pmc->counter, clear HW
     // Clear event selectors in HW

  3. perf_put_guest_lvtpc()
     // Restore host PMI vector

  4. perf_put_guest_context()
     // Return PMU control to host perf subsystem
```

## PMI (Performance Monitoring Interrupt) Delivery

### Emulated Path

```
Hardware counter overflow
  → Host perf PMI handler
  → perf_event overflow callback
  → kvm_perf_overflow(perf_event, ...)
    → __kvm_perf_overflow(pmc, in_pmi=true)
      → Set bit in pmu->global_status
      → kvm_make_request(KVM_REQ_PMI, vcpu)

  [On next KVM_RUN]
  → kvm_pmu_deliver_pmi(vcpu)
    → kvm_apic_local_deliver(APIC_LVTPC)
```

### Mediated Path

```
Hardware counter overflow (guest counter in real HW)
  → Hardware PMI (NMI or interrupt)
  → kvm_handle_guest_mediated_pmi()
    → kvm_get_running_vcpu()
    → kvm_make_request(KVM_REQ_PMI, vcpu)

  [On VM-Exit or next KVM_RUN]
  → kvm_pmu_deliver_pmi(vcpu)
    → kvm_apic_local_deliver(APIC_LVTPC)
```

## Event Filtering

KVM supports per-VM PMU event filters via the `KVM_SET_PMU_EVENT_FILTER` ioctl. The filter specifies allowed or denied events using masked event selectors.

Both modes check `pmc_is_event_allowed()` against the filter. The enforcement differs:

- **Emulated**: denied events are simply not programmed into perf (the counter stays idle)
- **Mediated**: denied events have their ENABLE bit stripped from `eventsel_hw` or `fixed_ctr_ctrl_hw`, so hardware won't count even though the MSR is loaded

## PEBS (Precise Event-Based Sampling)

PEBS support is primarily tied to the emulated PMU path, where perf events with `attr.precise_ip` are used. The emulated path handles PEBS through:
- `pmc_get_pebs_precise_level()` -- determines precision level
- Cross-mapped counter detection for PEBS counters
- `GLOBAL_STATUS_BUFFER_OVF` bit management

Mediated PMU interactions with PEBS require additional hardware support (PEBS output to DS area) and are an area of ongoing development.

## Module Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `enable_pmu` | true | Enable/disable vPMU entirely |
| `enable_mediated_pmu` | true | Enable/disable mediated PMU (falls back to emulated if disabled or unsupported) |

Both are read-only after module load (`0444` permissions).

## Key Source Files

| File | Contents |
|------|----------|
| `arch/x86/include/asm/kvm_host.h` | `struct kvm_pmc`, `struct kvm_pmu` definitions |
| `arch/x86/kvm/pmu.h` | `struct kvm_pmu_ops`, inline helpers, function declarations |
| `arch/x86/kvm/pmu.c` | Common PMU logic: MSR handling, reprogramming, mediated load/put, event filtering |
| `arch/x86/kvm/vmx/pmu_intel.c` | Intel PMU ops: RDPMC dispatch, LBR, mediated load/put, PEBS |
| `arch/x86/kvm/svm/pmu.c` | AMD PMU ops |
| `arch/x86/kvm/vmx/vmx.c` | `vmx_recalc_pmu_msr_intercepts()`, VMCS PMU field setup |

## See also

- [cmp-emulated-vs-mediated-pmu](../comparisons/cmp-emulated-vs-mediated-pmu.md) -- Side-by-side comparison of emulated vs mediated PMU
- [cmp-emulated-vs-mediated-pmu-zh](../comparisons/cmp-emulated-vs-mediated-pmu-zh.md) -- 模拟 PMU 与中介 PMU 对比分析（中文版）
- [kvm-cpu-virtualization](kvm-cpu-virtualization.md) -- VCPU lifecycle, VMCS, VM-Entry/Exit
- [kvm-interrupt-virtualization](kvm-interrupt-virtualization.md) -- APIC emulation, PMI delivery via LVTPC
- [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md) -- Progressive VM-Exit elimination
- [kvm-performance-tuning](kvm-performance-tuning.md) -- General KVM performance optimization
