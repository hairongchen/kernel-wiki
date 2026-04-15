---
type: source
created: 2026-04-10
updated: 2026-04-10
sources: [bytedance-solution-vmexit-code]
tags: [vm-exit, pv-ipi, posted-interrupt, kvm, vmx, patch, pi-desc, passthrough-ipi, bytedance]
title: "[RFC] KVM: X86: implement Passthrough IPI"
---

# Source: [RFC] KVM: X86: implement Passthrough IPI (ByteDance Patch)

| Field | Value |
|-------|-------|
| Author | dengqiao.joey (Qiao Deng), Yang Zhang — ByteDance |
| Date | September 5, 2020 |
| Mailing list | linux-kernel (lore.kernel.org) |
| Base kernel | Linux 5.0 |
| Status | RFC, not merged upstream |
| Files modified | 14 files, +434/-28 lines |

## Summary

This is the full kernel patch implementing ByteDance's **Passthrough IPI** (PV IPI) mechanism for KVM on x86. The patch paravirtualizes the IPI sending process by exposing Posted-Interrupt (PI) descriptors directly to the guest, allowing it to set PIR bits and write physical ICR without VM Exit/Entry overhead. This reduces IPI cost from **7,000 cycles to 2,000 cycles** (vs 1,500 cycles on bare metal host).

## Mechanism

### Core Idea

1. **Expose PI_DESC to guest**: KVM allocates pages for `pi_desc` structures (one per vCPU, 64 bytes each) and maps them into guest address space via a private memory slot
2. **Expose physical ICR**: Guest writes to a new `MSR_KVM_PV_ICR` (0x4b564d06) which KVM routes directly to `kvm_x2apic_msr_write()` for the real APIC ICR
3. **Guest-side IPI path**: Guest sets PIR bits in the target vCPU's PI_DESC, sets the ON (Outstanding Notification) bit, then writes physical ICR with notification vector and destination pCPU — all without trapping to hypervisor

### Data Structures

```c
union pvipi_msr {
    u64 msr_val;
    struct {
        u64 enable:1;    /* Guest sets to enable PV IPI */
        u64 reserved:7;
        u64 count:4;     /* Number of pages for pi_desc */
        u64 addr:51;     /* Base GPA of pi_desc pages (page-aligned) */
        u64 valid:1;     /* Hypervisor sets when pi_desc is ready */
    };
};
```

- `MSR_KVM_PV_IPI` (0x4b564d05): Control MSR for PV IPI enable/disable and pi_desc address
- `MSR_KVM_PV_ICR` (0x4b564d06): Replacement ICR MSR for SIPI/NMI/INIT (legacy path)
- `KVM_FEATURE_PV_IPI` (12): New CPUID feature bit

### Guest-Side Implementation (arch/x86/kernel/kvm.c)

The guest-side `kvm_setup_pv_ipi2()` function:
1. Reads `MSR_KVM_PV_IPI` to get pi_desc address from hypervisor
2. Maps pi_desc pages via `ioremap_cache()`
3. Writes enable bit back via `wrmsrl(MSR_KVM_PV_IPI, ...)`
4. Replaces APIC IPI functions (`send_IPI`, `send_IPI_mask`, etc.)
5. Replaces ICR read/write functions

The fast-path `kvm_send_ipi()`:
```
1. x2apic_wrmsr_fence()
2. Set PIR bit in target vCPU's pi_desc (test_and_set_bit)
3. Set ON bit in target pi_desc (test_and_set_bit)
4. Read nv (notification vector) and ndst (destination pCPU) from pi_desc
5. Write physical ICR via native_x2apic_icr_write()
```

If the PIR bit was already set, or ON was already set, the function returns early — the interrupt is already pending.

### Host-Side Implementation (arch/x86/kvm/vmx/vmx.c)

**PI_DESC management**:
- Changed `pi_desc` from embedded struct to pointer (`struct pi_desc *pi_desc`)
- `pi_desc_setup()`: Maps guest-physical pi_desc pages into host kernel via `kvm_vcpu_gfn_to_page()`
- PI_DESC pages allocated via `PVIPI_PAGE_PRIVATE_MEMSLOT` (new private memslot)
- Supports up to 128 vCPUs (2 pages × 64 pi_descs/page)

**MSR interception control**:
- When PV IPI is enabled, **disables MSR interception** for `X2APIC_MSR(APIC_ICR)` — guest ICR writes go directly to hardware without VM Exit
- When disabled (e.g., for migration), re-enables ICR interception

**CPUID**: Advertises `KVM_FEATURE_PV_IPI` only when APICv is active and x2APIC is enabled on the host.

## Mailing List Discussion

**Wanpeng Li** (Sept 7, 2020): Noted the idea is not new (references ACM paper doi:10.1145/3381052.3381317). Expressed security concerns — suitable for private cloud.

**Yang Zhang** (Sept 21, 2020): Reported huge improvement in daily ByteDance business including **TikTok**, especially under heavy futex workloads. Proposed turning the feature off by default for upstream acceptance. No response from maintainers (Paolo, Radim) recorded in this thread.

## Security Considerations

The patch explicitly acknowledges: "it may increase the risk in the system since the guest could decide to send IPI to any processor." The guest has direct access to:
- PI_DESC of all vCPUs (can set arbitrary PIR bits)
- Physical ICR write (can send IPI to any physical CPU)

This is acceptable in private cloud (ByteDance's use case) but not for multi-tenant public cloud.

## Key Themes

1. **Zero VM Exit IPI**: Eliminates both the ICR write VM Exit and the KVM emulation overhead
2. **Guest-driven posted interrupt**: The guest directly manipulates PI_DESC — a reversal of the normal host-controlled PI flow
3. **x2APIC requirement**: Relies on x2APIC mode where APIC ID equals vCPU ID for simple addressing
4. **Limited scalability**: Current implementation supports max 128 vCPUs (2 pages of pi_descs)
5. **Not upstream**: Security concerns prevented mainline acceptance; remains a private-cloud optimization

## Relationship to Other Sources

- Implements the "NoExit PVIPI" technique described in [src-minimizing-vmexits-pv-ipi-passthrough-timer](src-minimizing-vmexits-pv-ipi-passthrough-timer.md)
- Code-level detail for the IPI fastpath described at architecture level in [src-bytedance-solution-vmexit](src-bytedance-solution-vmexit.md)
- Built on Posted-Interrupt infrastructure described in [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md)

## See also

- [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md)
- [kvm-cpu-virtualization](../entities/kvm-cpu-virtualization.md)
- [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md)
