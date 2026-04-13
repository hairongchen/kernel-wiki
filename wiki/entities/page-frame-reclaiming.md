---
type: entity
created: 2026-04-08
updated: 2026-04-09
sources: [understanding-the-linux-kernel, professional-linux-kernel-architecture]
tags: [page-reclaiming, swap, oom-killer, lru, memory-management, lumpy-reclaim]
---

# Page Frame Reclaiming

The **Page Frame Reclaiming Algorithm (PFRA)** is the kernel subsystem responsible for recovering page frames when physical memory runs low. It decides which pages to evict from memory -- writing them to swap or simply discarding them -- to make room for new allocations. The PFRA in Linux 2.6.11 uses a dual-LRU scheme, reverse mapping for efficient page table teardown, a swap subsystem for anonymous page persistence, and an OOM killer as a last resort.

## Page classification

The PFRA classifies all page frames into four categories based on how they can be reclaimed:

| Category | Contents | Reclaim method |
|----------|----------|----------------|
| **Unreclaimable** | Free pages, reserved pages, kernel code/data, locked pages (`VM_LOCKED` / `PG_locked`), pages used by the slab allocator for non-shrinkable caches | Cannot be reclaimed. |
| **Swappable** | Anonymous pages (process heap, stack, private data), tmpfs pages | Must be written to a swap area before the frame can be freed. See [process-address-space](process-address-space.md). |
| **Syncable** | Clean file-backed pages in the [page-cache](page-cache.md), clean buffer pages | Can be reclaimed immediately (the data exists on disk). Dirty pages must be written back first, then reclaimed. |
| **Discardable** | Unused dentry cache entries, unused inode cache entries, empty slab pages | Can be freed immediately with no I/O. |

The PFRA preferentially reclaims discardable and syncable (clean) pages before touching swappable pages, since swap I/O is expensive.

## LRU lists

Each memory zone maintains two LRU (Least Recently Used) lists for pages that are candidates for reclamation:

- **Active list** (`zone->active_list`) -- Pages that have been recently accessed and are considered "working set." Pages here are protected from immediate reclamation.
- **Inactive list** (`zone->inactive_list`) -- Pages that have not been accessed recently and are candidates for reclamation. Pages are reclaimed from the tail of the inactive list.

Both lists are doubly-linked through the `page->lru` field. Pages with `PG_active` set are on the active list; those without are on the inactive list. The `PG_lru` flag indicates that a page is on one of the two lists.

### Page promotion and demotion

Pages move between the active and inactive lists based on access:

**Promotion (inactive -> active):**

- `mark_page_accessed()` is called when a page is accessed (read or write). On the first call, it sets `PG_referenced`. On the second call (if `PG_referenced` is already set), it promotes the page to the active list by calling `activate_page()`, which moves the page to the head of the active list and sets `PG_active`.
- This **two-touch** policy prevents pages that are accessed only once (e.g., sequential file reads) from polluting the active list.

**Demotion (active -> inactive):**

- `refill_inactive_zone()` (called by `shrink_zone()`) scans the tail of the active list and moves pages to the inactive list. It examines `PG_referenced`: if the flag is set, it is cleared and the page gets another chance (moved to the head of the active list); if clear, the page is demoted to the inactive list.
- The number of pages scanned and moved is controlled by a ratio that tries to keep the active and inactive lists in a balanced proportion relative to zone size.

### Interaction with page flags

| Flag | Set by | Cleared by | Effect |
|------|--------|------------|--------|
| `PG_referenced` | `mark_page_accessed()`, `page_referenced()` (scanning PTEs) | `refill_inactive_zone()`, `shrink_list()` | Protects page from eviction for one scan cycle. |
| `PG_active` | `activate_page()` | `refill_inactive_zone()` | Indicates membership in the active list. |
| `PG_lru` | `lru_cache_add()` / `lru_cache_add_active()` | `del_page_from_lru()` | Indicates membership in either LRU list. |

## Direct reclaim

When an allocation request fails and the caller can sleep (`__GFP_WAIT` is set), the allocator performs **direct reclaim** -- the calling process synchronously frees pages to satisfy its own allocation. The call chain:

```
__alloc_pages()
  -> try_to_free_pages(zones, gfp_mask)
    -> for each zone in priority order:
         shrink_zone(zone, sc)
           -> refill_inactive_zone()     // move pages from active to inactive
           -> shrink_cache()             // scan and reclaim from inactive list
             -> shrink_list()            // process individual pages
    -> shrink_slab()                     // shrink kernel caches (dcache, icache, etc.)
```

### shrink_zone()

`shrink_zone()` is the per-zone reclamation engine. It:

1. Calls `refill_inactive_zone()` to transfer pages from the active list to the inactive list, ensuring there are enough candidates.
2. Calls `shrink_cache()` to process pages on the inactive list.

It operates in **priority** levels (initially 12, decreasing to 0). At each priority level, the fraction of the zone's pages scanned increases. Lower priority means more aggressive scanning.

### shrink_cache() and shrink_list()

`shrink_cache()` takes batches of pages from the tail of the inactive list and passes them to `shrink_list()`, which processes each page:

1. **If the page is locked (`PG_locked`) or under writeback (`PG_writeback`)**: Skip it (or wait for writeback if in synchronous mode).
2. **Check references**: Call `page_referenced()`, which scans all PTEs mapping the page (via reverse mapping) to check and clear hardware-set accessed bits. If the page was recently referenced, move it back to the active list (promote).
3. **If the page is dirty**:
   - For file-backed pages: call `writepage()` to initiate I/O. The page is then marked `PG_writeback` and will be freed after I/O completes on a subsequent pass.
   - For anonymous pages: the page must be added to the swap cache and written to the swap area.
4. **Unmap the page**: Call `try_to_unmap()` to remove all page table entries pointing to this page (via reverse mapping). If any mapping is still pinning the page, reclamation fails for this page.
5. **If the page is clean, unmapped, and unreferenced**: Remove it from the page cache (or swap cache) via `remove_mapping()` and free the page frame to the buddy system.

## kswapd: background reclamation

**kswapd** is a per-NUMA-node kernel thread (`kswapd0`, `kswapd1`, ...) that performs background page reclamation. It sleeps until woken by the page allocator when any zone's free pages fall below `pages_low`.

### Operation

```
kswapd()
  loop:
    sleep until woken (zone free pages < pages_low)
    for each zone in the node:
      if zone free pages < pages_high:
        balance_pgdat() -> shrink_zone() + shrink_slab()
    continue until all zones have free pages >= pages_high
    go back to sleep
```

kswapd applies the same `shrink_zone()` / `shrink_slab()` logic as direct reclaim, but proactively and in the background. By maintaining free pages between `pages_low` and `pages_high`, kswapd reduces the frequency of costly direct reclaim in allocation paths.

### Watermark-driven behavior

The zone watermarks (see [memory-management](memory-management.md)) directly control kswapd:

- **`pages_high`**: Target. kswapd reclaims until free pages reach this level.
- **`pages_low`**: Trigger. kswapd is woken when free pages drop below this.
- **`pages_min`**: Emergency. Allocations below this threshold are only allowed for `PF_MEMALLOC` or `__GFP_HIGH` callers.

## Reverse mapping

To reclaim a page, the PFRA must invalidate every page table entry (PTE) that maps it. Without reverse mapping, the kernel would have to scan every process's entire page table to find mappings -- an O(n * m) operation (n processes, m pages per process). Linux 2.6 uses **reverse mapping** to find all PTEs for a given page in O(number of actual mappings).

### Anonymous reverse mapping: anon_vma

Anonymous pages (heap, stack, COW-copied pages) use the `anon_vma` system:

- Each VMA that maps anonymous pages has an `anon_vma` pointer.
- The `anon_vma` structure contains a linked list of all VMAs that could potentially contain PTEs pointing to the anonymous pages originally created in this range.
- When `fork()` duplicates a VMA, the child's VMA is added to the same `anon_vma` list.
- `page->mapping` for an anonymous page points to its `anon_vma` (with the low bit set to distinguish it from a file `address_space`).

To find all PTEs mapping an anonymous page, `try_to_unmap_anon()`:

1. Follows `page->mapping` to the `anon_vma`.
2. Walks the list of VMAs.
3. For each VMA, computes the virtual address from `page->index` and the VMA's `vm_start`/`vm_pgoff`.
4. Walks the page tables of `vma->vm_mm` to find and clear the PTE.

### File-backed reverse mapping: i_mmap

File-backed pages use the `address_space->i_mmap` priority search tree:

- Every VMA that maps a file region is inserted into the file's `address_space->i_mmap` tree, keyed by the file offset range.
- Given a page's `page->index` (file offset), `try_to_unmap_file()` queries the priority search tree to find all VMAs whose mapped range includes that offset.
- For each matching VMA, it computes the virtual address and walks the page tables to find and clear the PTE.

The priority search tree enables efficient queries even when many VMAs map overlapping file ranges (e.g., shared libraries mapped by hundreds of processes).

### try_to_unmap()

`try_to_unmap()` is the top-level reverse mapping function. It:

1. Determines if the page is anonymous (check `PageAnon()`, which tests the low bit of `page->mapping`) or file-backed.
2. Calls `try_to_unmap_anon()` or `try_to_unmap_file()` accordingly.
3. For each PTE found:
   - Acquires the page table lock (`page_table_lock` of the owning mm).
   - Clears the PTE.
   - Flushes the TLB entry.
   - Decrements `page->_mapcount`.
   - For anonymous pages, installs a **swap entry** in the PTE (encoding the swap device and offset) so the page can be swapped in later.
   - For file-backed pages, the PTE is simply cleared -- the data can be re-read from the file on fault.
4. Returns `SWAP_SUCCESS` if all mappings were removed, `SWAP_AGAIN` if some were skipped (locked mm, etc.), or `SWAP_FAIL` if the page cannot be unmapped.

## Swap subsystem

The swap subsystem provides persistent storage for anonymous pages that are evicted from memory. Without swap, anonymous pages would be unreclaimable, and memory pressure could only be relieved by evicting file-backed pages.

### Swap areas and swap_info_struct

A swap area is a disk partition or file dedicated to holding swapped-out pages. Each swap area is described by a `struct swap_info_struct`:

| Field | Description |
|-------|-------------|
| `swap_file` | The `struct file` for the swap file/partition. |
| `swap_map` | Array indexed by swap slot number. Each entry is a reference count: the number of processes whose PTEs contain swap entries pointing to this slot. When zero, the slot is free. |
| `flags` | `SWP_USED` (active), `SWP_WRITEOK` (writable), etc. |
| `pages` | Total number of usable slots. |
| `max` | Maximum slot number. |
| `prio` | Priority (higher priority areas are used first). |
| `cluster_nr` / `cluster_next` | Sequential allocation clustering for disk locality. |

Up to `MAX_SWAPFILES` (typically 32) swap areas can be active simultaneously.

### Swap extents

For swap files on filesystems (as opposed to raw partitions), `swap_info_struct` maintains a list of **swap extents** mapping contiguous ranges of swap slots to contiguous disk blocks. This allows the swap code to compute disk block numbers without going through the filesystem's block mapping code, enabling efficient direct I/O to the swap area.

### Swap cache (swapper_space)

The **swap cache** is a page cache for swap, using a dedicated `address_space` called `swapper_space`. Pages in the swap cache are indexed by their swap entry (encoding device and slot).

The swap cache serves multiple purposes:

1. **Write coalescing**: If a page is written to swap and then faulted back in before other processes swap it in, the swap cache prevents redundant I/O.
2. **Race avoidance**: When multiple processes share a swapped page (via COW `fork()`), the swap cache ensures only one copy of the page exists in memory as they fault it in concurrently.
3. **Writeback tracking**: A page remains in the swap cache while writeback is in progress (`PG_writeback`). It is removed from the swap cache once writeback completes and no PTEs reference it.

### Swap-out flow

When `shrink_list()` decides to swap out an anonymous page:

1. `add_to_swap()` allocates a swap slot and adds the page to the swap cache.
2. `try_to_unmap()` replaces all PTEs with swap entries.
3. `swap_writepage()` writes the page to the swap area.
4. On I/O completion, if `page->_mapcount` is -1 (no PTEs) and the swap cache is the sole reference, the page is freed.

### Swap-in flow

See `do_swap_page()` in [process-address-space](process-address-space.md).

### Swap token

The **swap token** is an anti-thrashing mechanism. Under heavy swap load, multiple processes may continuously swap each other's pages out, making no progress. One process at a time holds the token (stored in `swap_token_mm`), acquired via `grab_swap_token()` on swap faults. The token holder's pages are protected from eviction -- `page_referenced()` always returns true for its pages. The token has a timeout (`SWAP_TOKEN_TIMEOUT` jiffies) and is released when the timeout expires or the holder exits, ensuring at least one process can build a working set and make forward progress.

## OOM killer

When all reclamation efforts fail -- direct reclaim, kswapd, and slab shrinking cannot free enough pages -- the kernel invokes the **OOM (Out of Memory) killer** to sacrifice a process and reclaim its memory.

### Invocation chain

```
__alloc_pages()
  -> after all reclaim attempts exhausted
  -> out_of_memory()
    -> select_bad_process()
      -> for each process: badness(p)    // compute score
    -> oom_kill_process(selected)
      -> send SIGKILL to the selected process and its threads
```

### badness() scoring

`badness()` computes a numeric score for each candidate process. The process with the highest score is killed. The scoring heuristic considers:

1. **Base score**: The process's total virtual memory size (`mm->total_vm`). Larger processes score higher -- killing them frees more memory.
2. **CPU and runtime penalty**: Long-running processes score lower (they have invested more computation, so killing them is more costly). `p->start_time` is considered.
3. **Child process bonus**: Processes with many children score higher, because killing the parent often causes the children to be reaped too.
4. **Nice value adjustment**: Processes with positive nice values (low priority) score higher -- they are considered less important.
5. **Superuser discount**: Processes running as root (with `CAP_SYS_ADMIN` or equivalent) get their score divided by 4, since root processes are often system-critical.
6. **Direct hardware access**: Processes with `CAP_SYS_RAWIO` get their score divided by 4, since they may be performing raw I/O that cannot be easily interrupted.
7. **OOM adjustment**: The per-process `/proc/[pid]/oom_adj` value (set via `oom_adj`) provides a manual multiplier. A value of `OOM_DISABLE` (-17) makes the process completely immune.

The goal is to kill the process that frees the most memory while causing the least disruption.

### After the kill

When the selected process receives SIGKILL:

1. The process is marked with `TIF_MEMDIE`, granting it `PF_MEMALLOC`-like access to memory reserves so it can complete its exit path.
2. As the process exits, `exit_mmap()` unmaps all VMAs and frees all page tables and page frames.
3. The freed memory becomes available for pending allocations.

The OOM killer is a last resort. Proper system sizing, swap configuration, and memory resource limits (`RLIMIT_AS`, `RLIMIT_RSS`, cgroup memory limits in later kernels) should prevent OOM from being triggered in normal operation.

## Slab cache shrinking

In addition to reclaiming page cache and anonymous pages, the PFRA also shrinks kernel slab caches via `shrink_slab()`.

### Mechanism

`shrink_slab()` iterates over a list of registered **shrinker** callbacks (registered via `set_shrinker()`). Each shrinker represents a kernel cache that can release memory on demand. Key shrinkers include:

- **`shrink_dcache_memory()`** -- Reclaims unused dentry cache entries. Dentries on the `dentry_unused` list (those with zero reference count) are freed, which in turn releases their associated inodes.
- **`shrink_icache_memory()`** -- Reclaims unused inode cache entries after their dentries are freed.
- **`mb_cache_shrink_fn()`** -- Shrinks the extended attribute cache.

Each shrinker callback is called with two modes:

1. **Query mode** (`nr_to_scan == 0`): Returns the number of reclaimable objects.
2. **Shrink mode** (`nr_to_scan > 0`): Frees up to `nr_to_scan` objects and returns the number remaining.

`shrink_slab()` distributes the shrinking effort across all registered shrinkers proportionally to their reclaimable counts, scaling by the same priority level used for `shrink_zone()`.

After a shrinker frees objects, the slab allocator may find that entire slabs become empty (`slabs_free` list in the slab cache). These empty slabs' page frames are returned to the buddy system via `slab_destroy()`, completing the reclamation.

## Reclamation priority and scanning

The PFRA uses a priority system (values 12 down to 0) to control scanning aggressiveness. At priority 12 (default), only 1/4096th of each zone's pages are scanned; as reclaim fails, priority decreases until priority 0 scans all pages. This progressive approach avoids unnecessary scanning while guaranteeing thorough reclamation under severe pressure.

## Enhancements in Linux 2.6.24

### Lumpy Reclaim

**Lumpy reclaim** (new in 2.6.24) extends `shrink_inactive_list()` to improve the allocation of higher-order page blocks. When the standard LRU-ordered scanning fails to produce contiguous free regions, lumpy reclaim switches to a physically-contiguous scanning mode:

`isolate_lru_pages()` in `ISOLATE_BOTH` mode selects a target page, then scans the surrounding pageblock pulling all LRU pages (active or inactive) into the reclaim batch. These are processed through `shrink_page_list()`, creating contiguous free blocks for buddy coalescing. Complementary to [page mobility grouping](../concepts/concept-page-mobility.md): mobility prevents fragmentation proactively, lumpy reclaim resolves it reactively.

### Swap Token Enhancements

The swap token mechanism (described above) was enhanced with adaptive timeout. The token timeout starts at `SWAP_TOKEN_TIMEOUT` and adjusts based on swap activity -- increasing under heavy thrashing to give the holder more time to build a working set, decreasing when swap pressure is low. The token is now checked in `page_referenced_one()` rather than in the main reclaim path, providing more precise protection.

## See also

- [memory-management](memory-management.md) -- Physical memory allocator, zones, watermarks, buddy system
- [process-address-space](process-address-space.md) -- Page fault handling, swap-in via do_swap_page()
- [page-cache](page-cache.md) -- File-backed page caching and writeback
- [concept-copy-on-write](../concepts/concept-copy-on-write.md) -- COW and its interaction with swap and anonymous pages
- [virtual-filesystem](virtual-filesystem.md) -- Dentry and inode caches targeted by slab shrinking
- [concept-page-mobility](../concepts/concept-page-mobility.md) -- Page mobility grouping for anti-fragmentation
