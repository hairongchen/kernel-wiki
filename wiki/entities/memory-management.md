---
type: entity
created: 2026-04-08
updated: 2026-04-09
sources: [understanding-the-linux-kernel, professional-linux-kernel-architecture]
tags: [memory-management, buddy-system, slab-allocator, zones, slub, page-mobility, numa]
---

# Memory Management

Linux physical memory management in kernel 2.6.11 is organized as a three-layer allocator stack: the **buddy system** allocates contiguous page frames, **memory zones** partition physical memory by hardware constraints, and the **slab allocator** provides efficient object-level caching on top of the page allocator. This page covers the full physical allocation subsystem, from individual page frame descriptors up through `vmalloc()` and memory pools.

## Page frame descriptors

Every physical page frame in the system is represented by a `struct page` descriptor stored in the `mem_map` array. Key fields include:

- **`flags`** -- Page status flags (zone ID, PG_locked, PG_dirty, PG_slab, PG_reserved, etc.) encoded as bit positions.
- **`_count`** -- Reference count. When it drops to -1, the frame is free. Managed via `get_page()`/`put_page()`.
- **`_mapcount`** -- Number of page table entries mapping this frame, used by the reverse mapping subsystem (see [page-frame-reclaiming](page-frame-reclaiming.md)).
- **`lru`** -- Links the page into the active or inactive LRU list for page reclamation.
- **`mapping`** / **`index`** -- For pages in the [page-cache](page-cache.md), `mapping` points to the owning `address_space` and `index` is the page offset within that mapping.
- **`private`** -- Used by the [page-cache](page-cache.md) to point to `buffer_head` chains for block-level tracking.

The `struct page` is kept small (around 32 bytes on 32-bit x86) because one exists per physical page frame -- on a 1 GB system with 4 KB pages, that is 262,144 descriptors.

## Memory zones

Physical memory is partitioned into **zones** to accommodate hardware constraints on DMA and kernel address mapping. Each zone is represented by a `struct zone` (historically `zone_t`).

### Zone types

| Zone | Physical range (x86) | Purpose |
|------|----------------------|---------|
| **ZONE_DMA** | 0 -- 16 MB | Pages addressable by legacy ISA DMA controllers (24-bit address bus). |
| **ZONE_NORMAL** | 16 -- 896 MB | Directly mapped into the kernel's linear address space at PAGE_OFFSET. The workhorse zone for most kernel allocations. |
| **ZONE_HIGHMEM** | Above 896 MB | Not permanently mapped into kernel virtual address space. Must be accessed via `kmap()`/`kmap_atomic()` temporary mappings. Only relevant on 32-bit systems with more than ~896 MB of RAM. |

The 896 MB boundary comes from the 1 GB kernel address space (3 GB/1 GB user/kernel split): after subtracting space for vmalloc, fixmaps, and other regions, approximately 896 MB remains for the direct linear mapping.

### Zone structure

Each `struct zone` contains:

- **`free_area[MAX_ORDER]`** -- The buddy system free lists, one per order (0 through 10).
- **`pages_min`**, **`pages_low`**, **`pages_high`** -- Watermark thresholds controlling page reclamation behavior.
- **`zone_pgdat`** -- Back-pointer to the parent `pg_data_t` node (relevant for NUMA).
- **`lock`** -- Spinlock protecting the buddy free lists.
- **`active_list`**, **`inactive_list`** -- LRU lists for the [page-frame-reclaiming](page-frame-reclaiming.md) algorithm.
- **`pageset`** -- Per-CPU hot/cold page caches (see below).

### Watermarks

The three watermark levels drive the reclamation machinery:

- **`pages_min`** -- Critical threshold. When free pages fall below this, the allocator operates in emergency mode (only `__GFP_HIGH` or PF_MEMALLOC callers may proceed). [kswapd](page-frame-reclaiming.md) is definitely running.
- **`pages_low`** -- Low threshold. When free pages drop below this, `kswapd` is woken to begin background reclamation.
- **`pages_high`** -- High threshold. `kswapd` keeps reclaiming until free pages reach this level, then goes back to sleep.

Watermarks are computed at boot based on zone size by `init_per_zone_pages_min()`.

## The buddy system

The buddy system is the core page frame allocator. It manages blocks of contiguous page frames in power-of-2 sizes called **orders**: order 0 is 1 page (4 KB), order 1 is 2 pages (8 KB), up to order 10 (MAX_ORDER - 1), which is 1024 pages (4 MB).

### Data structures

Each zone has an array `free_area[MAX_ORDER]` of `struct free_area`:

```
struct free_area {
    struct list_head free_list;   /* doubly-linked list of free blocks */
    unsigned long    nr_free;     /* number of free blocks at this order */
};
```

A block's first page has `PG_buddy` set in its flags and its `private` field stores the block's order.

### Allocation: `__alloc_pages()`

`__alloc_pages()` is the heart of the zoned page allocator. Its logic proceeds as follows:

1. **Zonelist traversal** -- The caller provides a `gfp_mask` that determines which zones are acceptable. The allocator walks a `zonelist` (ordered list of fallback zones) looking for a zone with enough free pages.
2. **Try the buddy free list** -- For each candidate zone, attempt to find a free block at the requested order. If the exact order is not available, take a block from a higher order and **split** it: the surplus halves are placed on the free lists of successively lower orders.
3. **Watermark check** -- If the zone's free pages are below `pages_low`, wake `kswapd` for background reclamation. If below `pages_min`, the allocation may fail (unless the caller has `__GFP_HIGH` or `PF_MEMALLOC`).
4. **Direct reclaim** -- If allocation fails and `__GFP_WAIT` is set, the allocator calls `try_to_free_pages()` to synchronously reclaim memory, then retries. See [page-frame-reclaiming](page-frame-reclaiming.md).
5. **OOM killer** -- As a last resort, the OOM killer is invoked to sacrifice a process and free memory.

The per-zone lock (`zone->lock`) is held during buddy list manipulation.

### Deallocation: `free_pages_bulk()`

When pages are freed, `free_pages_bulk()` performs **coalescing**:

1. Check if the freed block's **buddy** (the adjacent block at the same order that would combine to form a block at the next higher order) is also free.
2. If the buddy is free, remove it from its free list, merge the two blocks into a single block of the next higher order, and repeat at the next order.
3. If the buddy is not free (or the maximum order is reached), place the block on the appropriate free list.

The buddy of a block at address `addr` and order `n` is found by XORing bit `n` of the page frame number: `buddy_pfn = pfn ^ (1 << order)`. This arithmetic relationship is what gives the buddy system its name.

## Per-CPU page caches

To reduce zone lock contention and improve cache locality, each zone maintains **per-CPU hot/cold page caches** (`struct per_cpu_pageset`). Each per-CPU list tracks a `count` of cached pages, `low`/`high` watermarks, and a `batch` size for bulk transfers. **Hot pages** (recently freed, likely cache-warm) are preferred for most allocations; **cold pages** are preferred for DMA buffers.

When an order-0 allocation is requested, the allocator checks the current CPU's hot cache first. If empty, a `batch` of pages is refilled from the buddy system under the zone lock. When the cache exceeds `high` pages, a batch is returned. Single-page frees go to the hot cache; `__free_pages_cold()` targets the cold cache. This means most single-page operations never touch the zone lock.

## The slab allocator

The buddy system allocates whole page frames, but the kernel frequently needs much smaller objects (dozens to hundreds of bytes). Allocating a full page for a 64-byte `struct dentry` would waste enormous amounts of memory. The **slab allocator** provides efficient, cache-friendly object allocation on top of the buddy system.

### Core concepts

The slab allocator operates on three levels:

1. **Cache** (`kmem_cache_t` / `struct kmem_cache`) -- Manages objects of a single type and size. For example, there is a dedicated cache for `struct task_struct`, another for `struct dentry`, etc. Created via `kmem_cache_create()`.

2. **Slab** -- A contiguous group of page frames (one or more pages) carved into fixed-size object slots. Each slab is in one of three states:
   - **Full** -- All objects allocated.
   - **Partial** -- Some objects free, some allocated. Allocation preferentially targets partial slabs.
   - **Free (empty)** -- All objects free. These slabs can be returned to the buddy system under memory pressure.

3. **Object** -- An individual allocation unit within a slab.

Each cache maintains three lists: `slabs_full`, `slabs_partial`, and `slabs_free`. Allocation first checks `slabs_partial`, then `slabs_free` (promoting a slab to partial), and only if both are empty does it request new page frames from the buddy system to create a new slab.

### Per-CPU array_cache

Each slab cache has a **per-CPU `array_cache`** providing a lockless fast path for allocation and deallocation:

```
struct array_cache {
    unsigned int avail;    /* number of objects available */
    unsigned int limit;    /* max number to hold */
    unsigned int batchcount; /* bulk transfer size */
    unsigned int touched;  /* was this cache recently used? */
    void *entry[];         /* stack of pointers to free objects */
};
```

Allocation checks the local CPU's `array_cache` first. If it has available objects, one is popped in O(1) with no locking. If the array is empty, a `batchcount` of objects is transferred from the slab lists (under the cache's spinlock) to the per-CPU array. Similarly, frees push objects onto the per-CPU array; when it exceeds `limit`, a batch is drained back to the slab lists.

This design means the vast majority of slab allocations and frees proceed without taking any lock, which is critical for scalability on SMP systems.

### Slab coloring

Each slab applies a small byte offset (**color**) before the first object so that objects from different slabs are distributed across hardware cache sets, reducing conflict misses. Colors cycle through the available values; the number of colors depends on the unused space within a slab and the hardware cache line size.

### Constructors and destructors

`kmem_cache_create()` accepts optional constructor and destructor callbacks. The constructor runs once when objects are first created (not on every allocation), allowing expensive initialization (e.g., mutex setup inside `struct inode`) to be preserved across recycling. Destructors are rarely used in Linux 2.6.

### Slab descriptor

Each slab has a descriptor (`struct slab`) that tracks:

- **`list`** -- Links the slab into one of the cache's three lists (full/partial/free).
- **`s_mem`** -- Address of the first object in the slab.
- **`inuse`** -- Number of currently allocated objects.
- **`free`** -- Index of the first free object (free objects are linked by an implicit per-object index array, forming a free list within the slab).

For small objects, the slab descriptor is stored inside the slab itself (at the beginning or end). For large objects (slab size >= PAGE_SIZE / 8), it is stored in a separate `kmem_cache` to avoid wasting a large fraction of the slab.

## kmalloc() and kfree()

`kmalloc()` is the general-purpose kernel memory allocator, analogous to user-space `malloc()`. It uses a set of **size-class slab caches** with geometrically increasing sizes: 32, 64, 128, 256, 512, 1024, 2048, 4096, 8192, 16384, 32768, 65536, and 131072 bytes (128 KB). There are two parallel sets: one for normal allocations and one for DMA-safe allocations (using `GFP_DMA`).

```c
void *kmalloc(size_t size, gfp_t flags);
void kfree(const void *objp);
```

`kmalloc()` rounds up the requested size to the nearest size class and allocates from the corresponding slab cache. The returned memory is physically contiguous (since each slab is built from buddy-allocated pages). `kfree()` determines which slab cache owns the object (via the page descriptor's `PG_slab` flag and slab/cache pointers) and returns it.

The maximum `kmalloc()` size is 128 KB. For larger contiguous allocations, `__get_free_pages()` (buddy system) or `vmalloc()` must be used.

## vmalloc()

`vmalloc()` allocates memory that is **virtually contiguous** but may be **physically discontiguous**. It does this by:

1. Allocating individual page frames (possibly from different zones and non-adjacent physical addresses) via the buddy system.
2. Setting up kernel page table entries to map these scattered frames into a contiguous range within the **vmalloc area**: the virtual address range from `VMALLOC_START` to `VMALLOC_END` (typically just above the direct mapping, from around 0xC0000000 + 896 MB + 8 MB gap to near the top of the kernel address space).

```c
void *vmalloc(unsigned long size);
void vfree(const void *addr);
```

Each vmalloc allocation is tracked by a `struct vm_struct` (not to be confused with user-space `vm_area_struct`) containing the virtual start address, size, and an array of `struct page` pointers for the backing frames.

`vmalloc()` is slower than `kmalloc()` because it must manipulate page tables and can cause TLB misses. It is used when large, contiguous virtual regions are needed but physical contiguity is not required -- for example, loading kernel modules and `ioremap()`. Vmalloc mappings are written only into the **init_mm** master page table; other processes discover them lazily via `vmalloc_fault()`. See [process-address-space](process-address-space.md).

## Memory pools (mempool_t)

A **memory pool** (`mempool_t`) is a reserve of pre-allocated objects that guarantees a minimum number of allocations will succeed even under extreme memory pressure, used by subsystems that cannot tolerate failure (e.g., block I/O paths needing `struct bio`). Created via `mempool_create()`, the pool first attempts a normal allocation via its `alloc_fn` (typically `kmem_cache_alloc()`). If that fails, it dips into the reserve. If the reserve is also empty and the caller can sleep (`__GFP_WAIT`), it waits for objects to be returned. Memory pools sit on top of the normal allocator as a last-resort safety net.

## GFP flags

All allocation functions accept a `gfp_mask` (Get Free Pages mask) that controls allocator behavior. Flags fall into three categories:

### Zone modifiers

| Flag | Effect |
|------|--------|
| `__GFP_DMA` | Allocate from ZONE_DMA only. |
| `__GFP_HIGHMEM` | Allow allocation from ZONE_HIGHMEM. |

### Action modifiers

| Flag | Effect |
|------|--------|
| `__GFP_WAIT` | The allocator may sleep (enables direct reclaim and I/O). |
| `__GFP_HIGH` | Access emergency memory reserves (below `pages_min`). |
| `__GFP_IO` | The allocator may start disk I/O for reclaim. |
| `__GFP_FS` | The allocator may call into filesystem code during reclaim. |
| `__GFP_COLD` | Prefer cold pages (useful for DMA buffers). |
| `__GFP_NOWARN` | Suppress allocation failure warnings. |
| `__GFP_REPEAT` | Retry allocation, but may still fail. |
| `__GFP_NOFAIL` | Retry indefinitely -- allocation must not fail. |
| `__GFP_NORETRY` | Do not retry -- fail immediately if reclaim does not help. |

### Compound flags

| Flag | Composition | Typical use |
|------|-------------|-------------|
| `GFP_ATOMIC` | `__GFP_HIGH` | Interrupt handlers, spinlock-held code. Cannot sleep. |
| `GFP_KERNEL` | `__GFP_WAIT \| __GFP_IO \| __GFP_FS` | Normal kernel allocation in process context. May sleep. |
| `GFP_USER` | `__GFP_WAIT \| __GFP_IO \| __GFP_FS` | User-space page allocation. May sleep. |
| `GFP_DMA` | `__GFP_DMA` | ISA DMA buffer. Cannot sleep by itself; combine with others. |

The critical rule: **code that holds a spinlock or is in interrupt context must use `GFP_ATOMIC`** (or another mask without `__GFP_WAIT`), because sleeping could cause deadlock. Process-context code that can sleep should use `GFP_KERNEL` to give the allocator maximum flexibility for reclaim.

## Allocation flow summary

```
kmalloc(size, GFP_KERNEL)
  -> kmem_cache_alloc() on size-class slab cache
    -> check per-CPU array_cache (lockless fast path)
    -> if empty: refill from slab lists (cache spinlock)
      -> if no partial/free slabs: allocate pages from buddy
        -> __alloc_pages()
          -> try per-CPU page cache (no zone lock)
          -> if empty: refill from buddy free lists (zone lock)
          -> if low on memory: wake kswapd / direct reclaim / OOM
```

## Developments in Linux 2.6.24

The following enhancements appear in Linux 2.6.24 (documented by Mauerer).

### SLUB Allocator

Since Linux 2.6.22, **SLUB** is the default slab allocator, replacing the original SLAB implementation described above. SLUB simplifies the design:

- **No per-CPU queues or slab coloring** — reduces complexity and memory overhead.
- **Freelist embedded in objects** — free objects contain a pointer to the next free object directly within their memory, eliminating the separate per-slab free list array.
- **Metadata in `struct page`** — slab metadata (`freelist`, `inuse`, `objects` count) is stored in the page descriptor fields, rather than in a separate `struct slab`.
- **Cache merging** — caches with compatible size, alignment, and flags are transparently merged into a single `kmem_cache`, reducing the total number of caches.

SLUB's per-CPU fast path uses the page's freelist pointer directly: allocation pops from the freelist, deallocation pushes. When the per-CPU page is exhausted, a new partial slab is obtained from the node's partial list.

The original SLAB allocator remains available via `CONFIG_SLAB`, and a minimal debug allocator `CONFIG_SLOB` exists for embedded systems.

### Page Mobility and Anti-Fragmentation

Pages are classified by mobility type to reduce external fragmentation in the buddy system:

- `MIGRATE_UNMOVABLE` — kernel allocations that cannot be relocated
- `MIGRATE_RECLAIMABLE` — page cache and slab pages that can be freed
- `MIGRATE_MOVABLE` — user-space pages that can be physically relocated

Each buddy free_area maintains separate free lists per migration type. Allocations first try the matching type's list, falling back to others via `__rmqueue_fallback()`. The `ZONE_MOVABLE` pseudo-zone (configured via `kernelcore=` boot parameter) restricts a memory region to movable allocations only.

See [concept-page-mobility](../concepts/concept-page-mobility.md) for full details.

### NUMA Enhancements

The 2.6.24 kernel provides refined NUMA support:

- `pg_data_t` per node with per-node zone lists and fallback ordering
- Memory policies (`MPOL_BIND`, `MPOL_PREFERRED`, `MPOL_INTERLEAVE`) for controlling allocation placement
- Node distance matrix for topology-aware fallback decisions
- `zonelist` ordering options: node-based (prefer local node across all zones) vs. zone-based (prefer specific zone type across nodes)

## See also

- [page-frame-reclaiming](page-frame-reclaiming.md) -- How the kernel recovers page frames under memory pressure
- [process-address-space](process-address-space.md) -- User-space virtual memory management and page faults
- [page-cache](page-cache.md) -- File data caching built on top of the page allocator
- [concept-memory-addressing](../concepts/concept-memory-addressing.md) -- x86 address translation, paging, segmentation
- [concept-copy-on-write](../concepts/concept-copy-on-write.md) -- Lazy allocation strategy used extensively by the memory subsystem
- [concept-page-mobility](../concepts/concept-page-mobility.md) -- Page mobility types and anti-fragmentation in the buddy system
