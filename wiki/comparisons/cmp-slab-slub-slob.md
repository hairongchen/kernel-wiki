---
type: comparison
created: 2026-04-09
updated: 2026-04-09
sources: [understanding-the-linux-kernel, professional-linux-kernel-architecture]
tags: [memory-management, slab-allocator, slub, slob, kernel-allocator]
---

# SLAB vs. SLUB vs. SLOB Allocators

Linux provides three implementations of the kernel object caching layer (the "slab allocator" subsystem). All three sit between the buddy system page allocator and the `kmalloc()`/`kmem_cache_alloc()` API, but they differ significantly in complexity, memory overhead, scalability, and target workload. This page compares the three side by side.

## Background

The kernel frequently allocates and frees small, fixed-size objects (process descriptors, inodes, dentries, network buffers, etc.). Allocating a full page frame for each such object would waste enormous amounts of memory. The slab allocator subsystem solves this by carving page frames into fixed-size object slots, caching freed objects for reuse, and aligning allocations to hardware cache lines.

All three allocators present the same API to the rest of the kernel:

- `kmem_cache_create()` / `kmem_cache_destroy()` -- create and destroy object caches
- `kmem_cache_alloc()` / `kmem_cache_free()` -- allocate and free objects from a cache
- `kmalloc()` / `kfree()` -- general-purpose allocation using size-class caches

The allocator is selected at kernel build time via `CONFIG_SLAB`, `CONFIG_SLUB`, or `CONFIG_SLOB`.

## Summary comparison

| Dimension | SLAB | SLUB | SLOB |
|-----------|------|------|------|
| **Introduced** | Linux 2.0 (1996) | Linux 2.6.22 (2007) | Linux 2.6.16 (2006) |
| **Default since** | 2.0 -- 2.6.21 | 2.6.22 -- present | Never default |
| **Removed** | Removed in 6.5 (2023) | Current default | Removed in 6.4 (2023) |
| **Design philosophy** | Feature-rich, highly optimized caching | Simplified design, fewer data structures | Minimal footprint for tiny systems |
| **Target workload** | General-purpose servers and desktops | General-purpose (all workloads) | Embedded systems with very limited RAM |
| **Code complexity** | ~6,000 lines | ~4,000 lines | ~600 lines |
| **Memory overhead** | Highest (multiple queues, coloring metadata) | Moderate (metadata in struct page) | Lowest (no per-cache metadata) |
| **NUMA awareness** | Yes (per-node lists, alien caches) | Yes (per-node partial lists) | No |
| **Scalability (many CPUs)** | Good (per-CPU array_cache) | Better (lockless per-CPU fastpath via cmpxchg) | Poor (single global lock) |
| **Debugging support** | Moderate (red zones, poisoning) | Excellent (SLUB_DEBUG, /sys/kernel/slab/) | Minimal |
| **Config symbol** | `CONFIG_SLAB` | `CONFIG_SLUB` | `CONFIG_SLOB` |

## SLAB -- the original

The original SLAB allocator, introduced by Mark Hemment based on Jeff Bonwick's 1994 Solaris slab allocator paper, was the Linux kernel's default for over a decade.

### Architecture

SLAB uses a three-level structure:

1. **`kmem_cache`** -- one per object type, tracks all slabs for that type
2. **Slab** -- a contiguous group of pages carved into object slots, tracked by `struct slab` with a free object index array
3. **Object** -- individual allocation unit within a slab

Each cache maintains three doubly-linked lists: `slabs_full`, `slabs_partial`, and `slabs_free`.

### Per-CPU caching

Each `kmem_cache` has a **per-CPU `array_cache`** -- a LIFO stack of pointers to free objects. Allocation checks this stack first (no locking needed). When the stack is empty, a `batchcount` of objects is transferred from the slab lists under the cache spinlock. When the stack exceeds its `limit`, a batch is drained back.

On NUMA systems, SLAB additionally maintains **shared `array_cache`** pools and **alien caches** (per-remote-node arrays) to handle cross-node allocations efficiently.

### Slab coloring

SLAB offsets the first object within each slab by a varying amount (the "color") so that objects from different slabs map to different hardware cache lines, reducing conflict misses. The number of available colors depends on the leftover space in the slab.

### Slab descriptor

Each slab has a `struct slab` descriptor containing the free object linked list, the in-use count, and the address of the first object. For small objects, the descriptor is stored inside the slab pages; for large objects, it is stored externally in a separate cache.

### Strengths

- Mature, battle-tested, well-understood behavior
- Effective hardware cache coloring for cache-sensitive workloads
- Per-CPU and per-node caching for SMP/NUMA scalability

### Weaknesses

- Complex code with many data structures (array_cache, alien caches, slab descriptors, three-list management)
- Higher memory overhead from per-slab descriptors, free object arrays, and multiple queue levels
- Object metadata (free list indices) stored separately from objects, adding indirection
- Difficult to debug; limited introspection
- Cache proliferation: many separate caches for similar-sized objects waste memory

## SLUB -- the current default

SLUB (the "Unqueued Slab Allocator"), written by Christoph Lameter, replaced SLAB as the default in Linux 2.6.22. Its design goal was to reduce complexity and memory overhead while maintaining or improving performance.

### Architecture

SLUB eliminates several SLAB data structures:

- **No `struct slab`** -- slab metadata (`freelist`, `inuse`, `objects` count) is stored directly in the `struct page` descriptor, which exists already for every page frame. This eliminates the separate slab descriptor entirely.
- **No per-slab free object arrays** -- free objects contain an inline pointer to the next free object directly within their own memory (similar to how `malloc` implementations embed free list pointers). This removes the off-object metadata array.
- **No slab coloring** -- SLUB omits this optimization, relying instead on natural address variation and larger cache sizes in modern hardware to avoid systematic conflict misses.
- **No three slab lists** -- instead of full/partial/free lists, SLUB maintains only a partial list per node. Full slabs are not tracked (they are found again when objects are freed). Empty slabs are returned to the buddy system immediately.

### Per-CPU fastpath

SLUB's per-CPU path is simpler but equally fast:

1. Each CPU has a currently-active slab page (`cpu_slab`). Allocation pops from this page's freelist using a `cmpxchg` instruction -- completely lockless.
2. When the per-CPU page is exhausted, a new partial slab is obtained from the node's partial list (under the node lock).
3. If no partial slabs exist, new pages are allocated from the buddy system.
4. Deallocation pushes the object back onto the slab's freelist. If the slab was previously full, it is added to the node's partial list.

Since Linux 3.x, SLUB further optimized the fastpath with `this_cpu_cmpxchg_double()` to atomically update both the freelist pointer and a transaction ID, eliminating the need for disabling interrupts on the allocation fastpath.

### Cache merging

When `kmem_cache_create()` is called for a new object type, SLUB checks whether an existing cache with compatible size, alignment, and flags already exists. If so, the new cache is transparently aliased to the existing one. This reduces the total number of active caches and improves slab utilization. For example, multiple 64-byte object caches with the same flags will share a single underlying `kmem_cache`.

Cache merging can be disabled with the `slub_nomerge` boot parameter when debugging requires separate caches.

### Debugging and introspection

SLUB provides superior debugging facilities:

- **`/sys/kernel/slab/`** -- a sysfs directory exposing per-cache statistics (object counts, partial slab counts, allocation/free counts, cache hit rates)
- **SLUB_DEBUG** (`CONFIG_SLUB_DEBUG`) -- enables red-zoning (guard bytes around objects to detect overflows), poisoning (filling freed objects with a pattern to detect use-after-free), and object tracking (recording allocation/free call stacks)
- Boot parameters: `slub_debug=FZPU` selectively enables Sanity checks (F), Red-Zoning (Z), Poisoning (P), and User tracking (U) per cache or globally

### Strengths

- Simpler code, easier to maintain and audit
- Lower memory overhead (no separate slab descriptors, no free object arrays)
- Cache merging reduces total cache count and waste
- Lockless per-CPU fastpath with cmpxchg
- Excellent debugging and profiling via sysfs and SLUB_DEBUG
- Continuing upstream development (SLUB is the only allocator receiving active optimization)

### Weaknesses

- No slab coloring (minor impact on modern hardware with large, highly-associative caches)
- Inline free pointers can theoretically be exploited for use-after-free attacks (mitigated by CONFIG_SLAB_FREELIST_HARDENED in modern kernels, which XOR-encrypts the pointers)

## SLOB -- the minimal allocator

SLOB (Simple List of Blocks) was designed by Matt Mackall for systems with very limited memory (embedded, IoT, single-board computers with a few MB of RAM).

### Architecture

SLOB is radically simpler than both SLAB and SLUB:

- **No per-cache slab lists or per-CPU caches** -- all allocations within a size range share common page lists
- **Three global free lists** based on object size: small (< 256 bytes), medium (256 -- 1024 bytes), and large (1024 -- PAGE_SIZE bytes). Objects larger than PAGE_SIZE go directly to the buddy system.
- **First-fit allocation within pages** -- each page is treated as a heap. Free space is tracked by a simple linked list of free blocks within the page. Allocation walks the list looking for the first block large enough.
- **No object alignment guarantees** beyond natural alignment -- saves memory but may hurt performance on alignment-sensitive architectures

### Per-CPU behavior

SLOB has no per-CPU caching. All allocations and frees go through the global free lists, protected by a single spinlock per list. This makes SLOB unsuitable for SMP systems with significant concurrency.

### Strengths

- Extremely small code footprint (~600 lines vs. ~4000--6000 for SLUB/SLAB)
- Minimal memory overhead: no per-cache metadata, no per-CPU arrays, no slab descriptors
- Best memory utilization for tiny systems because objects are packed tightly with first-fit allocation (no internal fragmentation from size-class rounding)
- Low boot-time memory consumption

### Weaknesses

- Poor scalability: global locks, no per-CPU caching, O(n) first-fit search
- No NUMA awareness
- No debugging or introspection features
- Higher external fragmentation over time due to first-fit allocation
- No object constructors/destructors
- Not actively maintained -- removed from kernel in 6.4 (2023) along with SLAB

## Trade-off analysis

### Memory overhead vs. scalability

The three allocators represent points on a fundamental trade-off curve:

- **SLOB** minimizes memory overhead by eliminating all per-CPU and per-cache metadata, but at the cost of serialized access through global locks. Appropriate when total RAM is measured in single-digit megabytes and the system has one CPU.
- **SLAB** invests significant memory in per-CPU array_caches, alien caches, shared caches, slab descriptors, and free object index arrays. This investment pays off in reduced lock contention and faster allocation on SMP systems.
- **SLUB** finds a middle ground: it eliminates SLAB's multi-level queuing overhead while retaining per-CPU fastpaths. By reusing `struct page` fields for metadata, it avoids SLAB's separate descriptor overhead. Cache merging further reduces per-cache waste.

### Fragmentation behavior

- **SLAB** and **SLUB** both use fixed-size slots within slabs, so **internal fragmentation** comes from rounding up to the nearest size class (e.g., a 48-byte request served from a 64-byte slot wastes 16 bytes). SLUB's cache merging can reduce the number of size classes, potentially increasing internal fragmentation slightly but reducing external fragmentation from partially-filled caches.
- **SLOB** uses first-fit allocation with variable-sized blocks, so it has minimal internal fragmentation initially, but accumulates **external fragmentation** as the free list develops small, unusable gaps.

### NUMA behavior

- **SLAB** has the most elaborate NUMA support: per-node slab lists, alien caches for cross-node frees (avoiding remote node lock contention), and careful node-local allocation preference.
- **SLUB** has clean per-node partial lists and generally achieves good NUMA locality with less code. In practice, SLUB's NUMA behavior is comparable or better than SLAB's because the simpler design reduces the chance of objects being queued on the wrong node.
- **SLOB** has no NUMA awareness. All allocations come from whichever pages happen to be on the global free lists, with no effort to maintain node locality.

### Debugging and hardening

- **SLUB** is clearly the best for debugging: `/sys/kernel/slab/` provides runtime statistics, and `SLUB_DEBUG` offers red-zoning, poisoning, and allocation tracking without recompilation (controlled via boot parameters). Modern kernels also support `CONFIG_SLAB_FREELIST_HARDENED` (XOR-encrypted free pointers) and `CONFIG_SLAB_FREELIST_RANDOM` (randomized freelist order) for exploit mitigation.
- **SLAB** has basic red-zoning and poisoning but lacks the sysfs interface and boot-time tunability.
- **SLOB** has no debugging support.

### Current status (as of Linux 6.x)

As of Linux 6.5 (2023), both SLAB and SLOB have been removed from the kernel. **SLUB is the only remaining slab allocator**. The removal was driven by:

1. **Maintenance burden** -- supporting three allocators with identical APIs meant triplicated bug fixes and feature work
2. **SLUB superiority** -- SLUB matched or exceeded SLAB's performance in essentially all benchmarks while being simpler
3. **SLOB's niche disappeared** -- even very small embedded systems now have enough RAM and CPU that SLUB's overhead is negligible, and SLUB's debugging features are valuable during development
4. **Security hardening** -- concentrating hardening efforts (freelist randomization, pointer encryption) on a single allocator is more effective

## When each allocator was appropriate

| Scenario | Best choice | Rationale |
|----------|------------|-----------|
| General-purpose server/desktop (pre-6.4) | SLUB | Best balance of performance, memory efficiency, and debuggability |
| Legacy kernels (pre-2.6.22) | SLAB | Only option; mature and well-optimized |
| Tiny embedded system (< 8 MB RAM, single CPU, pre-6.4) | SLOB | Minimal memory and code footprint |
| NUMA server with many nodes (pre-6.5) | SLUB or SLAB | Both have NUMA support; SLUB simpler, SLAB more elaborate |
| Kernel debugging / development | SLUB | sysfs stats, SLUB_DEBUG, boot-time tunability |
| Any kernel >= 6.5 | SLUB | Only option remaining |

## See also

- [memory-management](../entities/memory-management.md) -- Physical memory management including buddy system and slab allocator details
- [concept-page-mobility](../concepts/concept-page-mobility.md) -- Page mobility and anti-fragmentation in the buddy system
- [page-frame-reclaiming](../entities/page-frame-reclaiming.md) -- How the kernel recovers memory under pressure
