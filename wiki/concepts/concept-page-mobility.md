---
type: concept
created: 2026-04-09
updated: 2026-04-09
sources: [professional-linux-kernel-architecture]
tags: [memory-management, fragmentation, page-mobility, buddy-system, migration]
---

# Page Mobility and Anti-Fragmentation

Page mobility grouping is an anti-fragmentation strategy introduced in Linux 2.6.24. It classifies pages by how easily they can be relocated or freed, then segregates allocations of different mobility types into separate regions of physical memory. This keeps movable pages together and unmovable pages together, preserving large contiguous free blocks for higher-order allocations.

## The Fragmentation Problem

The [memory-management](../entities/memory-management.md) buddy allocator can satisfy higher-order (multi-page) allocations only when contiguous free blocks of the requested size exist. Over time, as pages are allocated and freed in varying patterns, free pages become scattered among allocated pages. This is external fragmentation: total free memory may be sufficient, but no contiguous block large enough to satisfy the request is available.

Fragmentation affects any allocation of order > 0, but the impact is most visible for:

- **Hugepages** -- order-9 allocations (2 MB on x86-64) require 512 contiguous page frames
- **DMA buffers** -- hardware often requires physically contiguous memory
- **Kernel stacks and large slab objects** -- various internal allocations that need multi-page blocks

Because the kernel cannot move arbitrary pages (a page mapped by DMA hardware or used for kernel data structures has a fixed physical address), traditional buddy allocation offers no defense against fragmentation over long uptimes.

## Page Mobility Types

The anti-fragmentation strategy classifies every page allocation into one of several mobility types based on how the page can be handled when contiguous free space is needed:

- **`MIGRATE_UNMOVABLE`** -- Pages that cannot be moved to a different physical address. This includes most kernel data structures, DMA buffers, and the majority of `kmalloc()` allocations. Unmovable pages are the primary cause of long-term fragmentation because once allocated, they permanently pin their physical location.

- **`MIGRATE_RECLAIMABLE`** -- Pages that cannot be moved but can be freed on demand. Page cache pages backed by disk and slab caches with registered shrinkers fall into this category. When a contiguous region is needed, these pages can be evicted (written back if dirty, then discarded) to create free blocks.

- **`MIGRATE_MOVABLE`** -- Pages that can be physically relocated to a different address. User-space pages are the canonical example: the kernel copies the page contents to a new frame and updates the corresponding page table entry. Page cache pages without DMA mappings can also be moved by updating their `address_space` mapping. This is the most flexible category.

- **`MIGRATE_RESERVE`** -- An emergency fallback pool that is not subject to mobility grouping. When all other fallback options are exhausted, pages are taken from this reserve.

- **`MIGRATE_ISOLATE`** -- A special type used during memory hotplug operations. Pages are temporarily marked as isolated to prevent the allocator from handing them out while a memory section is being prepared for removal.

## Buddy System Integration

The anti-fragmentation mechanism is integrated directly into the buddy allocator. The `free_area` structure in each zone maintains separate free lists for each migration type:

```c
struct free_area {
    struct list_head free_list[MIGRATE_TYPES];
    unsigned long nr_free;
};
```

Each zone contains an array of `free_area` structures, one per order (0 through `MAX_ORDER - 1`). Within each order, pages are distributed across the per-migration-type free lists. When allocating, the buddy system first tries the free list matching the requested migration type. This naturally groups pages of the same mobility together in physical memory, creating large contiguous regions of each type.

When a block is split (because a smaller order is requested than the smallest available block), the resulting buddy fragments remain on the same migration type's free list, preserving the grouping.

## Fallback Mechanism

When the preferred migration type's free list is empty at the requested order, `__rmqueue_fallback()` steals pages from other migration types following a defined fallback order:

- **UNMOVABLE** falls back to: RECLAIMABLE, then MOVABLE, then RESERVE
- **RECLAIMABLE** falls back to: UNMOVABLE, then MOVABLE, then RESERVE
- **MOVABLE** falls back to: RECLAIMABLE, then UNMOVABLE, then RESERVE

The fallback logic tries to steal the **largest available block** from the fallback type and changes its migration type to match the requesting type. Taking a large block rather than a small one minimizes future cross-type contamination: one large steal is less damaging than many small ones scattered across different pageblocks.

## Pageblock Grouping

Physical memory is divided into pageblocks, each typically `2^(MAX_ORDER-1)` pages in size (4 MB on x86 with 4 KB pages). Every pageblock carries a migration type in per-pageblock metadata stored in the zone's `pageblock_flags` bitmap.

The functions `get_pageblock_migratetype()` and `set_pageblock_migratetype()` read and write this metadata. When the buddy allocator splits a large block, the resulting pieces inherit the migration type of the pageblock they reside in. When a fallback steal changes a block's type, the pageblock metadata is updated so future allocations from that region follow the new type.

This coarse-grained grouping (at 4 MB granularity) is the fundamental unit of anti-fragmentation. Within a single pageblock, all allocations should ideally share the same mobility type.

## GFP Flags and Mobility

The migration type for an allocation is derived from the GFP (Get Free Pages) flags passed to the allocator:

| GFP flag | Migration type | Typical users |
|----------|---------------|---------------|
| `__GFP_MOVABLE` | `MIGRATE_MOVABLE` | User-space pages, page cache |
| `__GFP_RECLAIMABLE` | `MIGRATE_RECLAIMABLE` | Slab caches with shrinkers (inode cache, dentry cache) |
| Neither flag | `MIGRATE_UNMOVABLE` | Kernel data structures, `kmalloc()`, DMA buffers |

This mapping means that callers do not need to interact with the migration type system directly. The existing GFP flag conventions automatically route allocations to the correct mobility group.

## ZONE_MOVABLE

`ZONE_MOVABLE` is a pseudo-zone that accepts only `MIGRATE_MOVABLE` allocations. It is configured at boot time via the `kernelcore=` parameter, which specifies how much memory to reserve for potentially unmovable allocations. All remaining memory is placed into `ZONE_MOVABLE`.

Because every page in `ZONE_MOVABLE` is guaranteed to be movable, the zone can always be fully compacted or have all its pages migrated away. This property is critical for:

- **Memory hotplug** -- a DIMM can be safely hot-removed only if all pages on it can be migrated elsewhere. `ZONE_MOVABLE` provides this guarantee.
- **Hugepage allocation** -- a zone with no unmovable pages can always be defragmented to produce contiguous blocks.

## Effectiveness and Limitations

Page mobility grouping significantly reduces fragmentation for higher-order allocations under typical workloads. By keeping unmovable pages confined to a subset of pageblocks, the remaining memory stays available for compaction and large contiguous allocations.

However, mobility grouping cannot eliminate fragmentation entirely:

- Unmovable allocations still fragment within their own pageblocks
- Fallback steals can contaminate movable regions with unmovable pages under memory pressure
- The system relies on allocation patterns matching the grouping granularity

Later kernels added complementary mechanisms. Linux 2.6.35 introduced **memory compaction**, which actively migrates movable pages to create contiguous free blocks, building directly on the mobility infrastructure.

## Relationship to Other Subsystems

- **Lumpy reclaim** (2.6.24) -- reclaims contiguous physical pages around a target page to create free blocks. Complementary to mobility grouping: lumpy reclaim works best when neighboring pages are reclaimable or movable. See [page-frame-reclaiming](../entities/page-frame-reclaiming.md).
- **Memory hotplug** -- `ZONE_MOVABLE` enables safe hot-removal because all pages in the zone can be migrated away before the hardware is unplugged.
- **Hugepages** -- benefit directly from reduced fragmentation. Order-9 allocations (2 MB) are far more likely to succeed when movable and unmovable pages are segregated. See [concept-memory-addressing](concept-memory-addressing.md).
- **[concept-copy-on-write](concept-copy-on-write.md)** -- COW pages are user-space pages and therefore classified as `MIGRATE_MOVABLE`, contributing to the effectiveness of the grouping strategy.

## See also

- [memory-management](../entities/memory-management.md)
- [page-frame-reclaiming](../entities/page-frame-reclaiming.md)
- [concept-memory-addressing](concept-memory-addressing.md)
- [concept-copy-on-write](concept-copy-on-write.md)
