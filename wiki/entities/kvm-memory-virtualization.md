---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [qemu-kvm-source-code-and-application]
tags: [kvm, memory-virtualization, ept, shadow-page-tables, memory-slots]
---

# KVM Memory Virtualization

Memory virtualization is one of the two core tasks of a hypervisor (alongside [kvm-cpu-virtualization](kvm-cpu-virtualization.md)). In QEMU/KVM, it involves translating between multiple layers of address spaces and efficiently mapping guest physical memory to host physical memory, either through hardware-assisted Extended Page Tables (EPT) or software-based shadow page tables.

## Address Translation Layers

A virtualized environment introduces extra layers of address translation beyond what a native system requires. Understanding these layers is essential to understanding KVM memory virtualization. See also [concept-memory-addressing](../concepts/concept-memory-addressing.md).

| Address Type | Abbreviation | Description |
|---|---|---|
| Guest Virtual Address | GVA | Address used by guest user-space and kernel code |
| Guest Physical Address | GPA | Physical address as seen by the guest OS |
| Host Virtual Address | HVA | Address within the QEMU process's virtual address space |
| Host Physical Address | HPA | Actual machine physical address |

The full translation chain is:

```
GVA → GPA → HVA → HPA
 │        │        │
 │        │        └── Host page tables (managed by host kernel)
 │        └── QEMU memory regions / KVM memory slots
 └── Guest page tables (managed by guest OS)
```

- **GVA to GPA**: The guest OS manages its own page tables. The guest kernel sets up CR3 to point to its page table root. From the guest's perspective, this works identically to a bare-metal system.
- **GPA to HVA**: QEMU maps guest physical memory into its own process address space using `mmap()`. KVM memory slots record this mapping. Each slot associates a range of guest physical addresses (base GFN + npages) with a host virtual address (the `userspace_addr` field).
- **HVA to HPA**: The host kernel's standard page tables handle this translation, just like any other user-space memory access.

With **EPT enabled** (the common case on Intel hardware), the GPA-to-HPA translation is performed directly in hardware, bypassing the need for software intervention on every guest memory access.

## QEMU Memory Model

QEMU models the guest's physical address space using a hierarchy of data structures that ultimately feed into KVM memory slots.

### MemoryRegion Tree

The `MemoryRegion` is QEMU's fundamental abstraction for describing address spaces. Memory regions form a tree, where each region has a type:

- **RAM**: Backed by host memory (allocated via `mmap()`). Represents guest RAM.
- **ROM**: Read-only memory regions, used for firmware.
- **MMIO**: Memory-mapped I/O regions with read/write callbacks that invoke device emulation.
- **Alias**: A window into another MemoryRegion, possibly at a different offset. Used to map the same backing memory at multiple guest physical addresses.
- **Container**: A grouping node that holds child regions. The root address space is typically a container.

Regions have priorities, so when regions overlap at the same GPA, the higher-priority region wins.

### FlatView

The MemoryRegion tree is a convenient modeling abstraction, but KVM needs a flat list of non-overlapping address ranges. QEMU "renders" the tree into a **FlatView**: a sorted array of `FlatRange` entries, each describing a contiguous, non-overlapping region of the guest physical address space.

When the MemoryRegion tree changes (e.g., a PCI BAR is remapped), QEMU regenerates the FlatView and computes the diff against the previous one.

### MemoryRegionSection and KVM Memory Slots

Each `FlatRange` maps to a `MemoryRegionSection`, which is then communicated to KVM via the `KVM_SET_USER_MEMORY_REGION` ioctl. This ioctl creates, modifies, or deletes a KVM memory slot. The pipeline is:

```
MemoryRegion tree
    → FlatView (rendered flat ranges)
        → MemoryRegionSection (per listener)
            → KVM_SET_USER_MEMORY_REGION ioctl
                → kvm_memory_slot (in-kernel)
```

## KVM Memory Slots

Inside the kernel, KVM tracks guest physical memory through `kvm_memory_slot` structures.

### kvm_memory_slot Structure

Key fields of `struct kvm_memory_slot`:

| Field | Description |
|---|---|
| `base_gfn` | Starting guest frame number (GPA >> PAGE_SHIFT) |
| `npages` | Number of pages in this slot |
| `userspace_addr` | Host virtual address (HVA) of the backing memory in QEMU's address space |
| `flags` | Flags such as `KVM_MEM_LOG_DIRTY_PAGES` and `KVM_MEM_READONLY` |
| `dirty_bitmap` | Bitmap tracking which pages have been written (when dirty logging is enabled) |

### kvm_memslots Container

All memory slots for a given address space are held in a `kvm_memslots` structure. This container includes a **generation counter** that is incremented on every modification. The generation counter is used with **RCU** (Read-Copy-Update) to allow lockless reads of memory slot information on the fast path (e.g., during EPT violation handling). Readers can safely access the current slot array without taking locks; writers create a new copy, update it, and swap it in atomically.

### KVM_SET_USER_MEMORY_REGION Processing

The `KVM_SET_USER_MEMORY_REGION` ioctl is handled by `__kvm_set_memory_region()` in the kernel. This function supports four operations:

1. **Create**: A new slot is added. KVM allocates the `kvm_memory_slot`, sets up the dirty bitmap if requested, and installs it in `kvm_memslots`.
2. **Delete**: An existing slot is removed (signaled by setting `memory_size` to 0). KVM removes the slot and zaps any associated shadow/EPT page table entries.
3. **Modify flags**: The slot's flags change (e.g., enabling dirty logging). KVM write-protects all pages in the slot if dirty tracking is being enabled.
4. **Move**: The slot's guest physical address range changes. Internally, KVM deletes the old slot and creates a new one.

## Extended Page Tables (EPT)

EPT is Intel's hardware-assisted mechanism for GPA-to-HPA translation, eliminating the need for shadow page tables. AMD's equivalent is called NPT (Nested Page Tables).

### EPT Structure

EPT uses a page-table-like structure with four levels (on x86-64): PML4, PDPT, PD, and PT. Each level maps progressively smaller regions of the guest physical address space. The EPT pointer (EPTP) is loaded into the VMCS and points to the root of the EPT hierarchy.

### EPT Violation Handling Flow

When the guest accesses a GPA that has no valid EPT mapping, the hardware triggers an **EPT violation**, causing a VM Exit. The handling flow is:

```
Guest accesses unmapped GPA
    → EPT violation (hardware)
        → VM Exit
            → handle_ept_violation()
                → kvm_mmu_page_fault()
                    → tdp_page_fault()
                        → gfn_to_pfn()
                            → __gfn_to_hva_many() [GFN → HVA via memory slots]
                            → get_user_pages_fast() [HVA → HPA/PFN via host page tables]
                        → __direct_map()
                            → mmu_set_spte() [installs EPT entry]
    → VM Resume
```

Key steps in detail:

- **gfn_to_pfn()**: Translates a guest frame number to a host page frame number. First looks up the GFN in the memory slots to find the corresponding HVA, then calls `get_user_pages_fast()` to pin the host page and obtain the physical frame number.
- **__direct_map()**: Walks the EPT hierarchy from the root, allocating intermediate page table pages as needed, and installs the final mapping via `mmu_set_spte()`.
- **mmu_set_spte()**: Sets the actual EPT page table entry, configuring permissions (read/write/execute) and the physical address.

### kvm_mmu_page

The `kvm_mmu_page` structure represents one page of shadow or EPT page table entries. Key fields include the GFN it maps, its level in the hierarchy, and a `role` field encoding its properties (level, direct vs. indirect, access permissions, etc.).

## Shadow Page Tables (Non-EPT Fallback)

On hardware without EPT/NPT support, KVM uses **shadow page tables** — a software technique where KVM maintains page tables that combine both guest and host translations into a single set of page tables loaded into CR3.

### How Shadow Page Tables Work

- The guest OS manages its own page tables (GVA → GPA), but the hardware never directly walks them.
- KVM intercepts guest page table modifications (via write-protection of guest PT pages) and constructs **shadow page tables** that map GVA → HPA directly.
- CR3 is loaded with the root of the shadow page table, not the guest page table.
- When the guest modifies its page tables, KVM must synchronize the shadow page tables accordingly.

### Shadow Page Fault Handling

When a shadow page fault occurs, KVM invokes `FNAME(page_fault)` (where FNAME is a macro that expands based on the paging mode — e.g., `paging64_page_fault`). This function:

1. Walks the guest page tables to find the GVA-to-GPA mapping.
2. Translates the GPA to an HPA via the memory slots and host page tables.
3. Builds or updates the corresponding shadow page table entry mapping GVA → HPA.

The `kvm_mmu_page` structure is shared between EPT and shadow page table modes, but in shadow mode the `role` field includes additional information such as whether the page is a "direct" mapping (for non-paging guests) or an "indirect" mapping (shadowing a guest page table page), along with the guest access bits.

Shadow page tables are significantly more complex and slower than EPT because every guest page table modification requires a VM Exit and synchronization.

## MMIO Handling

Guest accesses to memory-mapped I/O regions require special handling because they must be routed to device emulation rather than mapping real memory.

### MMIO SPTE Mechanism

When a guest accesses an MMIO address:

1. The first access causes an **EPT violation** because no EPT mapping exists for the MMIO GPA.
2. KVM detects that the GFN does not correspond to a RAM-backed memory slot.
3. Instead of mapping real memory, KVM installs a special **MMIO SPTE** — an EPT entry with reserved bits set in a specific pattern that marks it as MMIO.
4. Subsequent accesses to this GPA cause an **EPT misconfiguration** (not a violation), because the reserved bits violate EPT format rules.
5. `handle_ept_misconfig()` recognizes the MMIO SPTE pattern, extracts the GPA, and routes the access to KVM's **instruction emulation** engine.
6. The emulator decodes the faulting instruction, determines the MMIO address and data, and forwards the operation to the QEMU device model (via an exit to user-space or ioeventfd/irqfd).

This two-step approach (violation on first access, misconfiguration on subsequent accesses) avoids repeated memory-slot lookups for known MMIO addresses.

## Dirty Page Tracking

Dirty page tracking is essential for [vm-live-migration](vm-live-migration.md), where the hypervisor must know which guest pages have been modified so they can be re-transferred to the destination host.

### Write-Protection Based Tracking

The default dirty tracking mechanism works as follows:

1. When dirty logging is enabled for a memory slot (via `KVM_MEM_LOG_DIRTY_PAGES` flag), KVM **write-protects** all EPT entries for pages in that slot.
2. When the guest writes to a tracked page, an EPT violation occurs.
3. The fault handler marks the corresponding bit in the slot's `dirty_bitmap` and removes the write protection so subsequent writes proceed without faults (until the bitmap is harvested).
4. User-space (QEMU) calls `KVM_GET_DIRTY_LOG` to retrieve the bitmap. This ioctl returns the current dirty bitmap and clears it, re-applying write protection so the next round of tracking begins.

### Intel PML (Page Modification Logging)

PML is a hardware feature that reduces the overhead of dirty page tracking:

- The processor maintains a **PML buffer** (a 512-entry log in the VMCS) that records GPAs of pages modified by the guest.
- When the buffer fills, a VM Exit occurs, and KVM drains the buffer entries into the dirty bitmap.
- PML avoids the overhead of write-protecting pages and handling EPT violations for every dirty page, significantly improving performance during live migration of write-intensive workloads.

## MMU Notifiers

KVM must stay synchronized with the host kernel's memory management. If the host reclaims a page (e.g., due to swapping, migration, or memory pressure), any EPT/shadow entries pointing to that page must be invalidated.

KVM registers with the **MMU notifier** infrastructure in the host kernel. Key callbacks include:

- **kvm_mmu_notifier_invalidate_page()**: Called when the host invalidates a single page. KVM removes all shadow/EPT entries mapping to that page's HVA, forcing re-faulting on the next guest access.
- **kvm_mmu_notifier_invalidate_range_start/end()**: Called for range-based invalidations. KVM blocks new page-table installations during the invalidation window and removes affected entries.

This mechanism ensures that KVM never holds stale mappings to pages the host has reclaimed or remapped, maintaining memory safety even under host memory pressure.

## See also

- [kvm-cpu-virtualization](kvm-cpu-virtualization.md)
- [vm-live-migration](vm-live-migration.md)
- [concept-memory-addressing](../concepts/concept-memory-addressing.md)
- [memory-management](memory-management.md)
