---
type: entity
created: 2026-04-08
updated: 2026-04-08
sources: [understanding-the-linux-kernel]
tags: [page-cache, caching, radix-tree, buffer-cache, writeback]
---

# Page Cache

The **page cache** is the kernel's primary mechanism for caching disk data in memory. Virtually all reads from and writes to regular files, block devices, and memory-mapped files pass through the page cache. In Linux 2.6.11, the page cache subsumes the older buffer cache, unifying file data and block metadata caching under a single framework built around the `address_space` abstraction and radix tree indexing.

## The address_space object

An `address_space` is the bridge between a cached object (typically a file inode) and the set of page frames holding its data. Each inode that has cached pages has an associated `address_space` (stored in `inode->i_mapping`, which usually points to `inode->i_data`).

Key fields of `struct address_space`:

| Field | Description |
|-------|-------------|
| `host` | The owning inode (NULL for swap cache). |
| `page_tree` | Radix tree mapping page index (file offset / PAGE_SIZE) to `struct page` pointers. |
| `tree_lock` | Spinlock protecting `page_tree`. |
| `i_mmap` | Priority search tree of all VMAs that map this address_space (used for reverse mapping by the [page-frame-reclaiming](page-frame-reclaiming.md) subsystem). |
| `i_mmap_lock` | Spinlock protecting `i_mmap`. |
| `nrpages` | Number of cached pages. |
| `a_ops` | Pointer to `struct address_space_operations`. |
| `backing_dev_info` | Information about the underlying device (bandwidth, congestion state, read-ahead hints). |
| `private_list` | List of dirty `buffer_head` structures associated with this address_space. |

### Ownership model

Each `address_space` is typically associated with one inode, so there is a separate page cache tree per file. Block devices also have their own `address_space` (the block device inode's `i_mapping`), which caches metadata blocks (superblock, inode table blocks, bitmap blocks, indirect blocks) that are accessed standalone rather than as part of a file's data.

This per-inode organization means lookups are O(1) relative to the file: given a file and an offset, the kernel goes directly to the inode's `address_space` and looks up the page index in the radix tree.

## address_space_operations

The `struct address_space_operations` provides methods for transferring data between the page cache and the backing store. Each filesystem implements its own version. Key methods:

| Method | Purpose |
|--------|---------|
| `readpage(file, page)` | Reads a single page from the backing store into the given page frame. Called on cache miss. The filesystem submits the I/O (typically via `mpage_readpage()` or `block_read_full_page()`) and returns; the page is locked until I/O completes. |
| `readpages(file, mapping, pages, nr_pages)` | Batch version of `readpage()` for read-ahead. Reads multiple pages in a single I/O submission for efficiency. |
| `writepage(page, wbc)` | Writes a single dirty page back to the backing store. Called by the writeback threads or direct reclaim. |
| `writepages(mapping, wbc)` | Batch writeback of multiple dirty pages. |
| `prepare_write(file, page, from, to)` | Prepares a page for a partial write. For filesystems with block sizes smaller than PAGE_SIZE, this reads in the surrounding blocks so the partial write does not corrupt existing data. Allocates `buffer_head` structures if needed. |
| `commit_write(file, page, from, to)` | Completes a write after the data has been copied into the page. Marks the relevant buffers and the page as dirty. |
| `set_page_dirty(page)` | Marks a page as dirty (alternative to the default `__set_page_dirty_buffers()`). |
| `direct_IO(rw, iocb, iov, offset, nr_segs)` | Bypasses the page cache entirely for O_DIRECT I/O. |

## Radix tree indexing

The page cache uses a **radix tree** (`page_tree` in the `address_space`) to map page indices to page frame pointers. The radix tree is a multi-level trie with a branching factor of 64 (2^6) at each level, using 6 bits of the index per level.

### Properties

- **O(1) lookup** -- For a given tree height, the number of levels is fixed (at most 6 levels for a 32-bit index), so lookup time is bounded by a small constant.
- **Space efficiency** -- Internal nodes are allocated only when needed, so a sparsely populated file does not waste memory on empty slots.
- **Range queries** -- The tree supports efficient iteration over ranges of indices, used by `find_get_pages()` and friends for batched operations.
- **Page tagging** -- Radix tree nodes carry tag bits that propagate upward. Two tags are used:
  - **`PAGECACHE_TAG_DIRTY`** -- At least one page in this subtree is dirty.
  - **`PAGECACHE_TAG_WRITEBACK`** -- At least one page in this subtree is under writeback I/O.

  These tags enable efficient scans for dirty or writeback pages without visiting every entry. For example, `filemap_fdatawrite()` can quickly find all dirty pages by following only tagged paths through the tree.

## Page states

Pages in the page cache transition through states tracked by flags in `page->flags`:

| Flag | Meaning |
|------|---------|
| `PG_locked` | Page is locked for I/O. Readers call `lock_page()` (sleeping) or `wait_on_page_locked()` to wait. Set during `readpage()` and `writepage()`. |
| `PG_uptodate` | Page contents are valid. Set after successful read I/O. Checked before serving data to user space. |
| `PG_dirty` | Page has been modified in memory but not yet written to disk. Set by `set_page_dirty()` after write operations. |
| `PG_writeback` | Page is currently being written to disk. Set at writeback start, cleared on I/O completion. A page can be `PG_dirty` and `PG_writeback` simultaneously if it is dirtied again during writeback. |
| `PG_referenced` | Page has been recently accessed. Used by the [page-frame-reclaiming](page-frame-reclaiming.md) algorithm to decide promotion/demotion between LRU lists. |
| `PG_lru` | Page is on one of the zone's LRU lists (active or inactive). |

A page's typical lifecycle in the cache: allocated -> locked -> I/O submitted (readpage) -> uptodate + unlocked -> accessed (referenced) -> dirtied -> writeback -> clean.

## Buffer cache integration

In Linux 2.6, the old standalone buffer cache has been absorbed into the page cache. Disk blocks are still tracked individually via `struct buffer_head`, but these buffer heads are now **attached to page cache pages** rather than living in an independent cache.

### buffer_head structure

Each `buffer_head` tracks the state of a single disk block within a page:

| Field | Description |
|-------|-------------|
| `b_state` | State flags: BH_Uptodate, BH_Dirty, BH_Lock, BH_Mapped, BH_New, etc. |
| `b_page` | The page this buffer belongs to. |
| `b_blocknr` | The disk block number. |
| `b_bdev` | The block device. |
| `b_size` | Block size in bytes. |
| `b_data` | Pointer to the data within the page. |
| `b_this_page` | Circular linked list of all buffer_heads for a single page. |

For a 4 KB page on a filesystem with a 1 KB block size, there are 4 buffer_heads per page. The `page->private` field points to the first buffer_head in the circular list.

### When buffer_heads are used

Buffer heads are needed when:

- The filesystem block size is smaller than PAGE_SIZE, so individual blocks within a page may have different states (some uptodate, some not; some dirty, some clean).
- Metadata blocks (superblock, indirect blocks, directory blocks) are accessed by block number through the block device's own `address_space`.
- `prepare_write()` must read in existing block data around a partial-page write.

For simple cases where a page maps to contiguous blocks that are always read/written together, the `mpage` helpers (`mpage_readpage()`, `mpage_writepage()`) bypass buffer_head overhead and submit I/O directly using `bio` structures.

## Read path

### Cache hit

```
generic_file_read() / do_generic_file_read()
  -> find_get_page(mapping, index)    // radix tree lookup
  -> if found and PG_uptodate:
       copy_to_user() from the page    // fast path, no I/O
```

### Cache miss

```
find_get_page() returns NULL
  -> page_cache_alloc()                // allocate new page frame
  -> add_to_page_cache(page, mapping, index)  // insert into radix tree, set PG_locked
  -> mapping->a_ops->readpage(file, page)     // submit I/O to backing store
  -> wait_on_page_locked(page)                // sleep until I/O completes
  -> if PG_uptodate: copy_to_user()
```

`add_to_page_cache()` inserts the page into the `address_space`'s radix tree and increments the page's reference count. The page is added in a locked state (`PG_locked`); the `readpage()` method is responsible for unlocking it when I/O completes.

### Handling races

If two threads simultaneously fault on the same uncached page, the radix tree lock (`tree_lock`) serializes the insertions. The second thread finds the page already in the tree (inserted by the first) and simply waits for it to become uptodate, avoiding duplicate I/O.

## Write path

Writes to the page cache (for buffered I/O) proceed as:

```
generic_file_write()
  -> prepare_write(file, page, from, to)   // allocate buffers, read if partial page
  -> copy_from_user() into the page         // copy user data
  -> commit_write(file, page, from, to)     // mark buffers/page dirty
  -> balance_dirty_pages_ratelimited()      // throttle if too many dirty pages
```

After `commit_write()`, the page is marked dirty (`PG_dirty`) and the dirty tag is set in the radix tree. The actual disk write is deferred to the writeback subsystem.

For `O_SYNC` writes, `generic_file_write()` additionally calls `filemap_fdatawrite()` and `filemap_fdatawait()` after the write to flush the data to disk synchronously.

## Read-ahead

The kernel's read-ahead mechanism detects sequential access patterns and prefetches upcoming pages into the page cache before they are requested, converting small random reads into large sequential I/O.

### file_ra_state

Each `struct file` has a `file_ra_state` structure tracking the read-ahead state:

| Field | Description |
|-------|-------------|
| `start` | Start index of the current read-ahead window. |
| `size` | Current window size (in pages). |
| `ahead_start` | Start index of the next (ahead) window. |
| `ahead_size` | Size of the ahead window. |
| `prev_page` | Index of the most recently accessed page (for sequentiality detection). |
| `ra_pages` | Maximum read-ahead size (configurable via `backing_dev_info`). |

### Adaptive window sizing

The read-ahead algorithm adapts the window size based on observed access patterns:

1. **Initial read**: A small read-ahead window (e.g., a few pages) is submitted.
2. **Sequential access confirmed**: If the process reads pages sequentially, the window doubles on each pass, up to `ra_pages` (typically 128 KB, or 32 pages).
3. **Random access detected**: If the access pattern becomes non-sequential, read-ahead is disabled (the window collapses to the minimum).
4. **Ahead window**: When the process reaches the beginning of the ahead window, a new async read-ahead is triggered for the next batch. This pipelining ensures I/O is always in flight for sequential workloads.

Read-ahead is critical for throughput -- without it, each page fault would trigger a separate 4 KB disk read, dramatically under-utilizing disk bandwidth.

## Writeback subsystem

Dirty pages must eventually be written back to disk to free memory and ensure data durability. Linux 2.6 uses **pdflush** kernel threads for background writeback.

### pdflush threads

The kernel maintains a dynamic pool of pdflush threads (between 2 and 8). They perform two tasks:

1. **Periodic writeback** -- Every `dirty_writeback_interval` centiseconds (default: 500 = 5 seconds), a pdflush thread wakes and writes back pages that have been dirty longer than `dirty_expire_interval` centiseconds (default: 3000 = 30 seconds).
2. **Background writeback** -- When the percentage of dirty pages in the system exceeds `dirty_background_ratio` (default: 10%), a pdflush thread begins writing dirty pages.

### Dirty page throttling

To prevent a single writer from filling all of memory with dirty pages, the kernel throttles write(2) callers:

- **`balance_dirty_pages_ratelimited()`** -- Called after each page write. Checks dirty page counts approximately every 32 pages (to amortize the cost).
- **`balance_dirty_pages()`** -- If the fraction of dirty pages exceeds `dirty_ratio` (default: 40%), the calling process is forced to write back some of its own dirty pages synchronously (via `writeback_inodes()`) until the dirty count drops below the threshold. This **throttling** prevents the dirty page count from growing without bound and ensures that heavy writers bear the cost of their own I/O.

### Writeback control: writeback_control

The `struct writeback_control` (wbc) passed to `writepage()` and `writepages()` tells the filesystem how to perform writeback:

| Field | Description |
|-------|-------------|
| `nr_to_write` | Number of pages to write (decremented as pages are written). |
| `sync_mode` | `WB_SYNC_NONE` (opportunistic) or `WB_SYNC_ALL` (must write all, for fsync). |
| `nonblocking` | If set, do not block on locked pages or request queues. |
| `for_reclaim` | Set when writeback is triggered by page reclaim rather than pdflush. |
| `range_start` / `range_end` | Byte range to write (for targeted writeback). |

## sync operations

Explicit synchronization system calls flush dirty data to persistent storage:

- **`sync()`** -- Writes all dirty pages and metadata system-wide. Calls `wakeup_bdflush()` (pdflush) and `sync_inodes()` / `sync_filesystems()`.
- **`fsync(fd)`** -- Writes all dirty pages and metadata for a specific file. Calls `filemap_fdatawrite(mapping)` to submit all dirty pages, then `filemap_fdatawait(mapping)` to wait for I/O completion. Also syncs the inode metadata via `write_inode_now()`.
- **`fdatasync(fd)`** -- Like `fsync()` but skips metadata that is not needed for subsequent data reads (e.g., `st_atime`, `st_mtime`). Only flushes metadata if the file size changed or blocks were allocated.

### filemap_fdatawrite() and filemap_fdatawait()

These are the workhorses of synchronous writeback:

- `filemap_fdatawrite(mapping)` -- Iterates over all pages with the `PAGECACHE_TAG_DIRTY` tag in the radix tree and calls `writepage()` for each. Uses `WB_SYNC_ALL` mode.
- `filemap_fdatawait(mapping)` -- Iterates over all pages with the `PAGECACHE_TAG_WRITEBACK` tag and waits (`wait_on_page_writeback()`) until each page's I/O completes.

Together, they guarantee that all data that was dirty at the time of the call has reached stable storage.

## Standalone block I/O caching

Filesystem metadata blocks (superblock copies, group descriptors, inode table blocks, block/inode bitmaps, indirect blocks) are often accessed by raw block number rather than file offset. These are cached in the **block device's own `address_space`** (`bdev->bd_inode->i_mapping`), indexed by block number converted to a page index.

The kernel function `__bread()` looks up a block in this address_space:

```
__bread(bdev, block, size)
  -> __getblk(bdev, block, size)       // find or create buffer_head
  -> if not BH_Uptodate:
       ll_rw_block(READ, 1, &bh)      // submit read I/O
       wait_on_buffer(bh)             // wait for completion
  -> return bh
```

This mechanism ensures that metadata I/O benefits from the same page cache infrastructure as data I/O -- caching, LRU management, and writeback all work uniformly.

## See also

- [memory-management](memory-management.md) -- Physical memory allocator providing the page frames used by the cache
- [page-frame-reclaiming](page-frame-reclaiming.md) -- Reclaiming page cache pages under memory pressure
- [process-address-space](process-address-space.md) -- Memory-mapped file I/O via page faults into the page cache
- [virtual-filesystem](virtual-filesystem.md) -- VFS layer and inode/file abstractions owning the address_space
- [block-layer](block-layer.md) -- Block I/O layer (bio, request queues) used by readpage/writepage
- [ext2-ext3](ext2-ext3.md) -- Filesystem-specific address_space_operations implementations
