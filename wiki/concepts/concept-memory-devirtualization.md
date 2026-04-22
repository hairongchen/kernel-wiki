---
type: concept
created: 2026-04-11
updated: 2026-04-11
sources: [kvm-devirt-kvmforum2022]
tags: [kvm, memory, ept, npt, devirtualization, paravirtualization, page-tables, partition-hypervisor]
---

# Memory De-virtualization

Memory de-virtualization is a technique that eliminates the stage-2 address translation layer (EPT on Intel, NPT on AMD) by using paravirtualized page table interfaces. Instead of the hardware walking two sets of page tables on every memory access, the guest operates directly on host-physical page frames in its own page tables. The hardware MMU sees only a single level of translation — identical to native execution.

## The Problem: EPT/NPT Overhead

Standard hardware-assisted memory virtualization uses two-stage address translation:

```
Standard VM (with EPT/NPT):

  Guest VA ──[guest PT walk]──> Guest PA ──[EPT/NPT walk]──> Host PA
                                            ↑
                                    Extra hardware walk
                                    (up to 24 memory refs
                                     for a TLB miss)
```

A TLB miss in a guest with 4-level paging and 4-level EPT requires up to **24 memory references** (each of the 4 guest page table levels requires its own 4-level EPT walk, plus the final EPT walk for the target page). This overhead is measurable even with large pages:

| Configuration | Cache line prefetch latency (ns) |
|---------------|-----------------------------------|
| Native host | 9.32 |
| VM with 1GB EPT pages | 11.1 (~19% overhead) |
| VM with 4KB EPT pages | 14.3 (~53% overhead) |

## The Solution: PV Page Tables

Memory de-virtualization eliminates EPT/NPT entirely. The guest's page tables contain host-physical frame numbers (PFNs) instead of guest-physical frame numbers (GFNs), so the hardware MMU can walk them directly:

```
Memory de-virtualized BM:

  Guest VA ──[guest PT walk]──> Host PA     (single-level, like native)
                ↑
        Guest page tables contain
        host PFNs, not guest GFNs
```

### Setup Phase

At BM startup, KVM performs three preparation steps:

1. **Pin all guest memory**: KVM statically pins every page of BM's guest memory, ensuring GFN-to-PFN mappings remain stable
2. **Build translation tables**: KVM creates two mapping tables:
   - **gfn-to-pfn**: Used when the guest writes page table entries (translate the guest's view of physical addresses to real physical addresses)
   - **pfn-to-gfn**: Used when the guest reads page table entries (translate real physical addresses back to the guest's view)
3. **Map tables into guest**: Both mapping tables are mapped into BM's address space

### PV Page Table Interfaces

The guest kernel is modified to use paravirtualized interfaces for page table operations:

**Writes (gfn→pfn translation):**

```
set_pgd(pgd, val)   →  translate GFNs in val to PFNs, then write
set_pud(pud, val)   →  translate GFNs in val to PFNs, then write
set_pmd(pmd, val)   →  translate GFNs in val to PFNs, then write
set_pte(pte, val)   →  translate GFNs in val to PFNs, then write
```

When BM writes a page table entry, the PV interface looks up the gfn-to-pfn table and substitutes the real PFN before writing to the page table. The hardware MMU then walks these entries and finds real physical addresses directly.

**Reads (pfn→gfn translation):**

```
pgd_val(pgd)   →  read entry, translate PFNs back to GFNs
pud_val(pud)   →  read entry, translate PFNs back to GFNs
pmd_val(pmd)   →  read entry, translate PFNs back to GFNs
pte_val(pte)   →  read entry, translate PFNs back to GFNs
```

When BM reads a page table entry (e.g., to check permissions or extract addresses for kernel use), the PV interface translates PFNs back to GFNs so the guest kernel sees the addresses it expects.

**CR3 handling:**

```
write_cr3(val)  →  translate the PGD's GFN to PFN, then write CR3
read_cr3()      →  read CR3, translate PFN back to GFN
```

**Page faults:**

MMIO regions (device register spaces) cannot be mapped into the guest's page tables because they require trap-and-emulate semantics. The guest page fault handler issues a **hypercall** when it encounters a fault on an MMIO address, allowing KVM to emulate the access.

### Architecture Diagram

```
                    BM (Guest)
    +------------------------------------------+
    |                                          |
    |  set_pgd ──→ gfn-to-pfn ──→ pgd entry   |
    |  set_pud     mapping         pud entry   |
    |  set_pmd     table           pmd entry   |
    |  set_pte                     pte entry   |
    |                                          |
    |  pgd_val ←── pfn-to-gfn ←── pgd entry   |
    |  pud_val     mapping         pud entry   |
    |  pmd_val     table           pmd entry   |
    |  pte_val                     pte entry   |
    |                                          |
    |  PF handler ──hypercall──→ KVM (MMIO)    |
    |                                          |
    +------------------------------------------+
                       |
              Guest page tables
           (contain host PFNs directly)
                       |
                   Hardware MMU
              (single-level walk)
```

## Performance Impact

Memory de-virtualization provides the **largest single performance improvement** among all KVM-devirt techniques:

| Optimization | Latency improvement vs VM |
|-------------|---------------------------|
| Interrupt + IPI + Timer Passthrough | 8% |
| **Memory devirtualization** | **14%** |
| DMA devirtualization | 2% |

Cache line prefetch latency with memory devirt (9.35 ns) is virtually identical to native host (9.32 ns) — a 0.3% difference versus 19%–53% for standard EPT.

## Trade-offs

**Advantages:**
- Eliminates all EPT/NPT translation overhead
- Cache line prefetch latency matches native
- Largest single contributor to overall performance gain

**Limitations:**
- Requires guest kernel modifications (PV interfaces)
- Guest memory must be statically pinned at startup (no balloon, no memory hot-plug — these are listed as future work)
- MMIO requires a hypercall (but MMIO is rare in steady-state operation)
- Increases guest kernel complexity (GFN↔PFN translation in every page table operation)

## Relationship to Other Approaches

Memory de-virtualization represents the opposite end of the spectrum from EPT/NPT:

| Approach | Guest modification | Translation overhead | Memory flexibility |
|----------|-------------------|--------------------|--------------------|
| Shadow page tables | None | None (single-level) | Full (dynamic) |
| EPT/NPT | None | High (two-level walk) | Full (dynamic) |
| Memory devirtualization | PV interfaces | None (single-level) | Limited (static pinning) |

Shadow page tables also achieve single-level translation, but require extensive VM Exit handling for every guest page table modification (`CR3` writes, `INVLPG`, etc.). Memory devirtualization avoids those exits by letting the guest handle translations itself via the PV interfaces.

## See also

- [kvm-devirt](../entities/kvm-devirt.md)
- [kvm-memory-virtualization](../entities/kvm-memory-virtualization.md)
- [concept-memory-addressing](concept-memory-addressing.md)
- [concept-hardware-virtualization](concept-hardware-virtualization.md)
- [src-kvm-devirt-kvmforum2022](../sources/src-kvm-devirt-kvmforum2022.md)
