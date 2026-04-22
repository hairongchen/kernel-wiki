---
type: source
created: 2026-04-11
updated: 2026-04-11
sources: [kvm-devirt-kvmforum2022]
tags: [kvm, virtualization, partition-hypervisor, devirtualization, vm-exit, bytedance, performance]
---

# KVM-devirt: Extending KVM to a Zero-overhead Partition Hypervisor

**Author:** Liang Deng, System Technologies and Engineering (STE) team, ByteDance
**Venue:** KVM Forum 2022
**Format:** Conference presentation (19 slides)

## Overview

This presentation introduces **KVM-devirt**, a ByteDance project that extends KVM into a zero-overhead partition hypervisor. The core idea is to eliminate *all* virtualization overhead — both VM Exits and additional address translations — after the guest kernel initialization phase. The guest (referred to as "BM" — bare metal) runs with performance indistinguishable from native hardware, while still benefiting from KVM's isolation and management capabilities.

## Motivation

Modern servers have rapidly increasing core counts (AMD Genoa: 384 cores, Intel Icelake: 224 cores), creating three problems for a single shared kernel:

1. **Scalability**: Lock contentions in file systems, network stack, scheduler, and memory management become bottlenecks at many-core scale
2. **Fault isolation**: If one application crashes the kernel, the entire server crashes
3. **Kernel customization**: A single kernel cannot fulfill diverse application requirements (different kernel configurations, boot parameters)

Standard KVM virtualization can partition the server with separate guest kernels, but introduces non-trivial overhead.

## KVM Virtualization Overhead (Three Categories)

### 1. VM Exits
- Timer virtualization
- IPI virtualization
- Virtio notifications
- HLT and MWAIT instructions
- CPUID instruction
- Host interrupts

### 2. Posted Interrupt Overhead
Although posted interrupts eliminate some VM Exits, overhead remains due to the complex hardware path.

### 3. Additional Address Translations
- Stage-2 address translations (EPT on Intel, NPT on AMD)
- DMA remap translation on IOMMU page tables

## KVM-devirt: Six Techniques

KVM-devirt combines six passthrough and de-virtualization techniques to achieve its goals:

### 1. Interrupt Passthrough

- **Does not use posted interrupts** — avoids the overhead of the complex PI hardware path
- Passes LAPIC registers (IRR, ISR, EOI) directly to the BM guest
- Uses **separate host and guest interrupt vectors** to avoid mixture
- Guest interrupts on guest cores are configured as IRQs, delivered directly to VMX non-root mode (external-interrupt-exiting = 0)
- Host interrupts on guest cores are configured as NMIs (cause VM Exit via NMI exiting)
- Re-triggers a self-IPI (IRQ) in the host NMI handler to solve IRQ-mask issues
- Sends self-IPIs at VM entry to inject virtual guest interrupts of emulated devices

### 2. Interrupt Remap (for passthrough devices)

- Does **not** use IOMMU interrupt posting capability
- VFIO fills the IRTE (Interrupt Remapping Table Entry) with the guest vector and physical APIC_ID of the core running the BM
- When BM changes virq-vcpu binding or guest vectors, VFIO updates the IRTE accordingly
- Device interrupts are delivered directly to the BM without any hypervisor involvement

### 3. IPI Passthrough

- KVM maps a **vAPIC_ID-to-pAPIC_ID mapping table** into BM at startup
- KVM updates this mapping whenever a vCPU thread migrates to a new physical core
- On the sending side: BM translates vAPIC_ID to pAPIC_ID and directly accesses the ICR to send IPI with guest vector
- On the receiving side: IPI arrives as IRQ and is directly delivered to BM without VM Exit

### 4. Timer Passthrough

- BM uses the **physical Local APIC timer** directly; Host Linux uses broadcast timer instead
- KVM maps the **TSC offset value** into BM and updates it whenever modified
- At `lapic_next_event`, BM subtracts the TSC offset from the guest TSC deadline and writes the result to MSR_TSCDEADLINE (using host time base)
- On LAPIC timer expiration, the timer interrupt (IRQ) is delivered directly to BM without VM Exit

### 5. Memory De-virtualization

- BM uses **only stage-1 page tables** — EPT/NPT is disabled
- At BM startup, KVM statically pins BM's guest memory and maps both gfn-to-pfn and pfn-to-gfn translation tables into BM
- BM uses PV interfaces for page table writes: `set_pgd`/`set_pud`/`set_pmd`/`set_pte` translate gfn→pfn before writing to page tables
- BM uses PV interfaces for page table reads: `pgd_val`/`pud_val`/`pmd_val`/`pte_val` translate pfn→gfn
- The hardware MMU directly uses BM's guest page tables (no two-level translation)
- MMIO traps are handled via hypercall in the guest page fault handler

### 6. DMA De-virtualization

- KVM statically pins BM's memory and maps gfn-to-pfn mappings into BM
- The passthrough device driver calls `dma_map` which translates GPA→HPA using gfn-to-pfn mappings before issuing the DMA request
- VFIO configures the IOMMU in **passthrough mode**, disabling DMA remap address translations
- Requires device driver modifications to ensure `dma_map` is invoked at PAGE_SIZE granularity

### 7. Virtio Notification Passthrough

- **Frontend→backend**: BM directly accesses the ICR to send an IPI (with host vector) to the host core — no VM Exit
- **Backend→frontend**: Host sends an IPI (with guest vector) to the guest core, delivered directly as IRQ to BM via the interrupt passthrough mechanism

### Other Optimizations

- CPU isolation and `nohz_full` to eliminate scheduler/tick VM Exits
- CPUID instruction handled in BM via **dynamic binary rewriting** (avoids CPUID VM Exit)
- Some host IPIs to guest cores are delayed to the next VM Exit
- HLT and MWAIT instruction passthrough

## Benchmark Results

### IPI Latency
- **Intel**: BM matches host latency, both significantly below VM. Flat scaling with concurrency
- **AMD**: BM matches host. Standard VM (swx2apic) degrades slightly; AVIC VM degrades dramatically at high concurrency (concurrency=9)

### Timer Latency
- **Both Intel and AMD**: BM approximately matches host. Standard VM is ~2x worse than host/BM

### Cache Line Prefetch Latency

| Configuration | Latency (ns) |
|---------------|-------------|
| Native Host | 9.32 |
| BM (KVM-devirt) | 9.35 |
| VM with 1GB EPT pages | 11.1 |
| VM with 4KB EPT pages | 14.3 |

Memory de-virtualization (eliminating EPT) nearly eliminates the address translation overhead.

### Real-World Application (BM vs VM)

| Optimization | End-to-end latency improvement vs VM |
|-------------|--------------------------------------|
| Interrupt + IPI + Timer Passthrough | 8% |
| Memory devirtualization | 14% |
| DMA devirtualization | 2% |
| ALL combined | 20%–30% |

### Real-World Application (BM vs Native Host)

| Scenario | Normalized Latency (lower = better) |
|----------|--------------------------------------|
| 1 partition — Native Host | 1.00 |
| 1 partition — BM | 1.01 |
| 4 partitions — Native Host | 1.00 |
| 4 partitions — BM | 0.91 |

With a single partition, BM has only 1% overhead vs native. With four partitions, BM is **9% faster** than native — the kernel scalability benefit of running smaller, isolated kernels outweighs the minimal remaining virtualization cost.

## Status and Future Work

**Status (as of 2022):**
- Supports both Intel and AMD platforms
- Supports both QEMU and Cloud-hypervisor as VMM

**Future work:**
- Kernel patches posted to upstream
- Live migration support
- virtio-balloon and memory hot plug support

## Key Themes

1. **Partition hypervisor philosophy**: Use virtualization hardware not for consolidation but for partitioning — giving each workload its own kernel while eliminating virtualization overhead
2. **Paravirtualization for de-virtualization**: PV interfaces (set_pgd/pte_val/vAPIC mapping) are used to *remove* layers of virtualization rather than add them
3. **Holistic approach**: All six overhead categories are addressed together — partial solutions leave significant residual overhead
4. **Scalability through isolation**: At 4 partitions, BM outperforms native because smaller kernels have fewer scalability bottlenecks

## Pages informed

- [kvm-devirt](../entities/kvm-devirt.md) — KVM-devirt entity page
- [concept-memory-devirtualization](../concepts/concept-memory-devirtualization.md) — Memory de-virtualization concept
- [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md) — Updated with KVM-devirt as next-generation approach
- [concept-exitless-timer](../concepts/concept-exitless-timer.md) — Updated with KVM-devirt timer passthrough context

## See also

- [src-bytedance-solution-vmexit](src-bytedance-solution-vmexit.md)
- [src-minimizing-vmexits-pv-ipi-passthrough-timer](src-minimizing-vmexits-pv-ipi-passthrough-timer.md)
- [src-lcna-co2012-sekiyama](src-lcna-co2012-sekiyama.md)
- [concept-exitless-timer](../concepts/concept-exitless-timer.md)
- [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md)
