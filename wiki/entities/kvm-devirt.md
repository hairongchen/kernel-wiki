---
type: entity
created: 2026-04-11
updated: 2026-04-11
sources: [kvm-devirt-kvmforum2022]
tags: [kvm, partition-hypervisor, devirtualization, vm-exit, bytedance, performance, paravirtualization]
---

# KVM-devirt

KVM-devirt is a ByteDance project that extends KVM into a **zero-overhead partition hypervisor**. Rather than using virtualization for server consolidation (running many VMs on one host), KVM-devirt uses it for **server partitioning** — dividing a large many-core server into isolated partitions, each running its own kernel with near-zero virtualization overhead. The guest partition is called "BM" (bare metal).

## Motivation

As server core counts grow (AMD Genoa: 384 cores, Intel Icelake: 224 cores), running a single Linux kernel across all cores encounters problems:

- **Scalability bottlenecks**: Lock contention in file systems, network stack, scheduler, and memory management
- **Fault isolation**: One application crashing the kernel takes down the entire server
- **Kernel customization**: Different applications may need different kernel configurations or boot parameters

KVM can partition the server with separate guest kernels, providing isolation and customization, but standard virtualization overhead is non-trivial. KVM-devirt eliminates that overhead.

## Architecture

KVM-devirt achieves zero overhead by eliminating both VM Exits and additional address translations after the guest kernel initialization phase. It combines six techniques:

```
+----------------------------------------------------+
|                  Server Hardware                    |
|  +------------------+  +------------------+        |
|  |   Guest Cores    |  |   Host Cores     |        |
|  |  (VMX non-root)  |  |  (VMX root)      |        |
|  |                  |  |                  |        |
|  |  BM (bare-metal  |  |  Host Linux     |        |
|  |   guest kernel)  |  |  + KVM          |        |
|  |                  |  |                  |        |
|  |  - Stage-1 PT    |  |  - Broadcast    |        |
|  |    only (no EPT) |  |    timer        |        |
|  |  - Physical      |  |  - VFIO/IOMMU  |        |
|  |    LAPIC timer   |  |    config       |        |
|  |  - Direct IPI    |  |                  |        |
|  |    via ICR       |  |                  |        |
|  +------------------+  +------------------+        |
+----------------------------------------------------+
```

### 1. Interrupt Passthrough

Standard approach: Guest interrupts cause VM Exit; hypervisor injects virtual interrupt on VM Entry.

KVM-devirt approach:
- LAPIC registers (IRR, ISR, EOI) passed directly to BM
- Guest and host use **separate interrupt vector ranges** to avoid conflicts
- Guest interrupts arrive as IRQs directly into VMX non-root mode (VMCS external-interrupt-exiting = 0)
- Host interrupts targeting guest cores arrive as NMIs (which do cause VM Exit)
- Host NMI handler re-triggers a self-IPI to handle the IRQ-mask issue
- Emulated device interrupts injected via self-IPI at VM Entry

This approach deliberately **avoids posted interrupts** because, although PI eliminates VM Exits, the complex hardware PI path still adds overhead.

### 2. Interrupt Remap (Passthrough Devices)

For VFIO passthrough devices:
- IOMMU interrupt posting is **not** used
- VFIO programs the IRTE directly with the guest interrupt vector and the physical APIC_ID of the core running BM
- Device interrupts arrive directly at BM as IRQs — no hypervisor involvement
- VFIO updates IRTE whenever virq-vcpu binding or guest vectors change

### 3. IPI Passthrough

- KVM maps a **vAPIC_ID-to-pAPIC_ID table** into BM's address space at startup
- Updated whenever a vCPU migrates to a different physical core
- BM's `send_IPI` function translates vAPIC_ID → pAPIC_ID, then directly accesses the ICR with guest vector
- Receiving core gets the IPI as an IRQ delivered directly to BM (via interrupt passthrough)

### 4. Timer Passthrough

- BM takes **exclusive ownership of the physical LAPIC timer** on its cores
- Host Linux falls back to broadcast timer on host cores
- KVM maps the **TSC offset** into BM and keeps it updated
- BM's `lapic_next_event` subtracts TSC offset from guest TSC deadline, writes to MSR_TSCDEADLINE
- Timer interrupt fires as a direct IRQ to BM — no VM Exit

### 5. Memory De-virtualization

The most impactful single optimization (14% latency improvement in real workloads):

- **EPT/NPT is disabled** — BM uses only stage-1 (native) page tables
- KVM statically pins all BM memory at startup
- KVM maps gfn-to-pfn and pfn-to-gfn translation tables into BM
- Guest kernel uses **PV page table interfaces**:
  - Write: `set_pgd`/`set_pud`/`set_pmd`/`set_pte` translate gfn→pfn
  - Read: `pgd_val`/`pud_val`/`pmd_val`/`pte_val` translate pfn→gfn
- Hardware MMU walks BM's page tables directly (single-level translation)
- MMIO handled via hypercall in the guest page fault handler

See [concept-memory-devirtualization](../concepts/concept-memory-devirtualization.md) for details.

### 6. DMA De-virtualization

- KVM maps gfn-to-pfn mappings into BM
- Device drivers use `phys_to_dma` to translate GPA→HPA before issuing DMA requests
- VFIO configures IOMMU in **passthrough mode** (DMA remap disabled)
- Requires device driver modifications for PAGE_SIZE granularity `dma_map` calls

### Additional Optimizations

- **CPU isolation + nohz_full**: Eliminates scheduler/tick interrupts on guest cores
- **Dynamic binary rewriting**: Handles CPUID instructions inside BM without VM Exit
- **Delayed host IPIs**: Some host IPIs to guest cores deferred until next VM Exit
- **HLT/MWAIT passthrough**: Guest can halt/wait without VM Exit

## Performance

### Micro-benchmarks

| Metric | Host | BM | VM |
|--------|------|----|----|
| IPI latency | Baseline | ~Host | ~2x Host (Intel), degrades with concurrency (AMD AVIC) |
| Timer latency | Baseline | ~Host | ~2x Host |
| Cache line prefetch | 9.32 ns | 9.35 ns | 11.1 ns (1GB EPT) / 14.3 ns (4KB EPT) |

### Real-World End-to-End Latency (vs VM)

| Optimization | Improvement |
|-------------|-------------|
| Interrupt + IPI + Timer Passthrough | 8% |
| Memory devirtualization | 14% |
| DMA devirtualization | 2% |
| All combined | 20%–30% |

### Partition Scaling (vs Native Host)

| Scenario | BM Normalized Latency |
|----------|----------------------|
| 1 partition | 1.01 (1% overhead) |
| 4 partitions | 0.91 (9% **faster** than native) |

The 4-partition result demonstrates the key insight: smaller kernels running on fewer cores avoid scalability bottlenecks, more than compensating for any residual virtualization cost.

## Platform Support

- **CPU**: Intel (VT-x) and AMD (SVM)
- **VMM**: QEMU and Cloud-hypervisor
- **Status** (as of KVM Forum 2022): Internal deployment at ByteDance; upstream patches planned but not yet merged
- **Future work**: Live migration, virtio-balloon, memory hot plug

## Relationship to Other Work

KVM-devirt builds on and extends several techniques documented elsewhere in this wiki:

- **Interrupt passthrough**: Extends Sekiyama's direct interrupt delivery (2012) with a more complete vector separation scheme
- **Timer passthrough**: Uses the same physical-LAPIC-timer approach as ByteDance's earlier RFC patches, but integrated into the full devirt framework
- **IPI passthrough**: Related to ByteDance's Passthrough IPI RFC (2020), here combined with the interrupt passthrough mechanism
- **Memory devirtualization**: A novel PV approach that goes beyond hardware-assisted two-stage translation

## See also

- [concept-memory-devirtualization](../concepts/concept-memory-devirtualization.md)
- [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md)
- [concept-exitless-timer](../concepts/concept-exitless-timer.md)
- [kvm-cpu-virtualization](kvm-cpu-virtualization.md)
- [kvm-memory-virtualization](kvm-memory-virtualization.md)
- [kvm-interrupt-virtualization](kvm-interrupt-virtualization.md)
- [kvm-performance-tuning](kvm-performance-tuning.md)
- [virtio-framework](virtio-framework.md)
- [analysis-vfio-device-passthrough](../analyses/analysis-vfio-device-passthrough.md)
- [src-kvm-devirt-kvmforum2022](../sources/src-kvm-devirt-kvmforum2022.md)
