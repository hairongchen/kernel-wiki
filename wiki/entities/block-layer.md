---
type: entity
created: 2026-04-08
updated: 2026-04-08
sources: [understanding-the-linux-kernel]
tags: [block-layer, io-scheduler, bio, block-devices]
---

# Generic Block Layer

The generic block layer is the kernel subsystem that manages I/O requests to block devices (disks, partitions, and similar random-access storage). It sits between the filesystem/page-cache layer above and the device drivers below, providing a uniform interface for submitting, merging, sorting, and dispatching I/O operations. It interacts with the [virtual-filesystem](virtual-filesystem.md) for file-level I/O and with the [device-driver-model](device-driver-model.md) for device management.

## Core Data Structures

### bio

The **`struct bio`** is the fundamental unit of block I/O. A bio describes a single I/O operation that transfers data between memory and a contiguous range of disk sectors.

Key fields:

- `bi_sector` -- starting sector number on the target device
- `bi_bdev` -- pointer to the target `block_device`
- `bi_rw` -- read/write direction and flags
- `bi_vcnt` -- number of segments in the scatter-gather list
- `bi_io_vec` -- pointer to an array of `struct bio_vec` elements, each describing one memory segment:
  - `bv_page` -- the page containing the data
  - `bv_offset` -- offset within the page
  - `bv_len` -- length of this segment
- `bi_idx` -- current index into `bi_io_vec` (for partial completion)
- `bi_size` -- remaining bytes to transfer
- `bi_end_io` -- completion callback function
- `bi_private` -- opaque data for the callback

The scatter-gather design means a single bio can describe data spread across multiple non-contiguous pages in memory, all targeting a contiguous range of sectors on disk. This is essential for efficient I/O with the page cache, where logically contiguous file data may reside in physically scattered pages.

### request

A **`struct request`** aggregates one or more bios for contiguous (or nearly contiguous) sectors into a single I/O operation that can be issued to the device driver as one command.

Key fields:

- `sector` -- starting sector
- `nr_sectors` -- total sector count
- `bio` -- linked list of bios making up this request
- `biotail` -- pointer to the last bio in the list
- `flags` -- request flags (`REQ_RW`, `REQ_BARRIER`, `REQ_FAILFAST`, etc.)
- `rq_disk` -- the target gendisk
- `elevator_private` -- opaque data for the I/O scheduler
- `deadline` -- expiration time used by the deadline scheduler

When a new bio arrives, the block layer attempts to **merge** it into an existing request if the sectors are adjacent. Merging reduces the number of separate I/O operations and is one of the primary performance optimizations in the block layer. A bio can be merged at the front (prepended) or back (appended) of an existing request.

### request_queue

Each block device has a **`struct request_queue`** that holds pending requests and controls how I/O is submitted to the device.

Key fields:

- `queue_head` -- list of pending requests
- `make_request_fn` -- function called for each new bio (default: `__make_request`, which applies the I/O scheduler). Can be overridden for stacking devices like RAID or device-mapper.
- `request_fn` -- function called to dispatch requests to the device driver (the "strategy routine")
- `elevator` -- pointer to the I/O scheduler state
- `queue_flags` -- flags including `QUEUE_FLAG_PLUGGED` (queue is plugged, accumulating requests)
- `max_sectors` -- maximum sectors per request
- `max_phys_segments`, `max_hw_segments` -- scatter-gather limits
- `bounce_gfp`, `bounce_pfn` -- bounce buffer parameters
- `nr_requests` -- maximum number of queued requests (congestion control)

## I/O Submission Path

The path from a filesystem `write()` or `read()` to the disk driver follows these steps:

1. **`submit_bio()`**: the entry point. The filesystem or page cache layer constructs a bio and passes it to `submit_bio()`, which sets the read/write flag and calls `generic_make_request()`.

2. **`generic_make_request()`**: validates the bio (checks that the sectors are within the device's range), handles partition remapping (adjusts the starting sector for partitions), and calls the queue's `make_request_fn`.

3. **`__make_request()`** (the default `make_request_fn`): this is where the I/O scheduler gets involved.
   - First, attempt to **merge** the bio into an existing request. The I/O scheduler provides `elevator_merge()` which checks for adjacent requests.
   - If merging succeeds, the bio is added to the existing request and no new request is created.
   - If merging fails, allocate a new `struct request` from the request queue's memory pool, initialize it with the bio, and insert it into the queue via `elevator_insert()`.
   - If the queue is unplugged, call `__generic_unplug_device()` to start dispatching.

4. **I/O scheduler**: sorts and orders requests according to its algorithm (see below). When the queue is unplugged, the scheduler moves requests from its internal data structures to the dispatch queue.

5. **`request_fn`**: the device driver's strategy routine. It dequeues requests from the dispatch queue and programs the hardware (DMA transfers, command registers, etc.).

## I/O Schedulers (Elevators)

Linux 2.6 provides four pluggable I/O schedulers that control the order in which requests are dispatched to the device driver. The choice of scheduler significantly impacts performance depending on the workload.

### Noop Scheduler

The simplest scheduler: a plain **FIFO** queue. New requests are appended to the tail. Adjacent bios are still merged, but no reordering is performed. Suitable for devices with no seek penalty (SSDs, flash, virtual disks) or when the device has its own sophisticated request reordering (hardware RAID controllers).

### Deadline Scheduler

Maintains two sorted queues (one for reads, one for writes) ordered by sector number, plus two FIFO queues ordered by submission time. Each request has an **expiration deadline**:

- **Read deadline**: 500 ms (default)
- **Write deadline**: 5000 ms (default)

Normal dispatch takes requests from the sorted queues (elevator order, minimizing seeks). However, if the oldest request in either FIFO has exceeded its deadline, it is dispatched immediately. This prevents **starvation**: without deadlines, a continuous stream of requests near the current head position could indefinitely postpone requests at distant sectors.

Reads have a shorter deadline than writes because reads are typically synchronous (the process is blocked waiting) while writes are usually asynchronous (the process has already continued).

### Anticipatory Scheduler

Extends the Deadline scheduler with a crucial optimization: after completing a read request, it **pauses briefly** (typically 6 ms) before dispatching the next request. The hypothesis is that the process that just submitted the read will soon submit another read for a nearby sector. If such a read arrives during the pause, it can be served immediately with a minimal seek instead of servicing a distant write and then seeking back.

This anticipation heuristic uses per-process I/O statistics to predict whether the process is performing sequential reads. If the process's access pattern is random, the scheduler skips the anticipatory pause. The anticipatory scheduler significantly benefits workloads with many sequential readers competing with writers.

### CFQ (Completely Fair Queuing) Scheduler

Assigns a **separate request queue to each process** (identified by the I/O context). The scheduler services these per-process queues in round-robin order, giving each process a **time slice** for dispatching. Within each queue, requests are sorted by sector for elevator ordering.

CFQ's goal is **fairness**: each process gets an equal share of disk bandwidth. This prevents a single I/O-intensive process from monopolizing the disk at the expense of interactive processes. CFQ was the default scheduler in many Linux 2.6 distributions due to its good interactive performance.

Key parameters:
- `quantum` -- maximum number of requests dispatched per time slice
- `queued` -- maximum queue depth per process
- Time slice duration is configurable

## Queue Plugging and Unplugging

**Plugging** is a mechanism to accumulate requests before dispatching them to the device, allowing better merging and sorting.

When a queue is **plugged** (`QUEUE_FLAG_PLUGGED` set), new requests are added to the queue but not dispatched. The queue stays plugged while the kernel expects more I/O to arrive soon (e.g., during a page cache readahead sequence).

**Unplugging** occurs when:

- A timer expires (`blk_unplug_timeout`, typically a few milliseconds)
- The queue reaches a threshold number of requests (`blk_unplug_threshold`)
- The kernel explicitly calls `blk_run_queue()` or `generic_unplug_device()`
- A process waits for I/O completion (no point accumulating more if we are already blocked)

When unplugged, the I/O scheduler's dispatch function is called, which moves requests from the scheduler's internal queues to the dispatch queue and invokes the driver's `request_fn`.

## Request Completion

When the device completes an I/O transfer, the driver calls:

1. **`end_that_request_first()`**: updates the request's position and count, completing individual bios by calling their `bi_end_io` callbacks. If the entire request is not yet complete (e.g., partial transfer), returns 1 and the driver continues the transfer.

2. **`end_that_request_last()`**: called when `end_that_request_first()` returns 0, indicating the request is fully complete. This removes the request from the queue and frees it.

The bio's `bi_end_io` callback notifies the upper layer (filesystem, page cache, or direct I/O) that the data transfer is complete. For reads, this typically involves unlocking pages and waking blocked processes. For writes, it decrements the page's writeback count.

## Disk and Partition Representation

### gendisk

**`struct gendisk`** represents a physical disk:

- `major`, `first_minor`, `minors` -- device number range
- `disk_name` -- human-readable name (e.g., "sda")
- `part` -- array of `hd_struct` pointers representing partitions
- `fops` -- pointer to `block_device_operations`
- `queue` -- pointer to the device's request queue
- `capacity` -- total size in sectors

The `add_disk()` function registers a gendisk with the block layer, making it visible in `/proc/partitions` and `/sys/block/`.

### hd_struct

**`struct hd_struct`** represents a partition:

- `start_sect` -- starting sector of the partition relative to the whole disk
- `nr_sects` -- size in sectors
- `partno` -- partition number
- `reads`, `writes`, `read_sectors`, `write_sectors` -- I/O statistics

When a bio targets a partition, `generic_make_request()` adds `hd_struct->start_sect` to the bio's starting sector to convert from partition-relative to disk-absolute addressing.

### block_device_operations

**`struct block_device_operations`** provides device-level callbacks:

- `open()` / `release()` -- called when the device is opened/closed
- `ioctl()` -- device-specific control commands (e.g., querying geometry)
- `media_changed()` -- check if removable media was changed (e.g., CD ejected)
- `revalidate_disk()` -- re-read partition table after media change

## Bounce Buffers

Some devices (notably ISA bus devices) can only perform DMA to a limited physical address range (below 16 MB for ISA). If a bio's pages reside in high memory or above the DMA-reachable range, the block layer allocates **bounce buffers** in low memory, copies the data (for writes), issues the I/O using the bounce buffer, and copies the result back (for reads). The `bounce_pfn` field in the request queue specifies the highest physical page frame number reachable by the device's DMA.

Modern devices with full 32-bit or 64-bit DMA capability do not need bounce buffers, and the kernel avoids the copy overhead for these devices.

## Barrier I/O

Disk drives and their caches may reorder writes for performance. When the filesystem requires strict write ordering (e.g., the journal commit block must hit disk after all journal data blocks), it issues a **barrier request** (flagged with `REQ_BARRIER`). The block layer ensures:

1. All previously submitted requests complete.
2. A cache flush command is sent to the drive.
3. The barrier request is submitted.
4. Another cache flush is sent after the barrier completes.

This guarantees that the barrier request's data is durable on the physical media before any subsequent writes, which is critical for journal integrity in filesystems like [ext2-ext3](ext2-ext3.md).

## See also

- [virtual-filesystem](virtual-filesystem.md)
- [ext2-ext3](ext2-ext3.md)
- [device-driver-model](device-driver-model.md)
- [page-cache](page-cache.md)
