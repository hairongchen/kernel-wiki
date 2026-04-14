---
type: source
title: "Understanding the Linux Kernel, 3rd Edition"
created: 2026-04-08
updated: 2026-04-08
sources: [understanding-the-linux-kernel]
tags: [reference-book, linux-2-6, kernel-internals, x86, o-reilly]
---

# Understanding the Linux Kernel, 3rd Edition

**Authors:** Daniel P. Bovet and Marco Cesati
**Publisher:** O'Reilly Media, November 2005
**Covers:** Linux kernel 2.6.11, x86 (80x86) architecture
**ISBN:** 978-0-596-00565-8

## Overview

A comprehensive, bottom-up guide to the Linux kernel's internal design and implementation. The book takes a hardware-aware approach, starting with x86-specific mechanisms (segmentation, paging, interrupts) and building up to portable subsystems (VFS, scheduler, memory management). Code from kernel 2.6.11 is dissected line by line, with simplified rewrites of hand-optimized sections for readability. The authors -- professors at the University of Rome "Tor Vergata" -- developed the material from a course encouraging students to read and modify kernel source.

## Chapter Summaries

### Ch 1: Introduction
Surveys Linux's Unix heritage, monolithic-with-modules architecture, and the major subsystems: process management, memory management, VFS, IPC, and networking. Introduces User Mode vs. Kernel Mode (Ring 0/3 on x86), kernel reentrancy, and the 3 GB user / 1 GB kernel virtual address space split on 32-bit x86. Key data structures introduced: [task_struct](../entities/process-management.md), [mm_struct](../entities/process-address-space.md), [inode](../entities/virtual-filesystem.md), [dentry](../entities/virtual-filesystem.md), [superblock](../entities/virtual-filesystem.md).

### Ch 2: Memory Addressing
Details x86 address translation: logical -> linear (segmentation) -> physical (paging). Linux uses a flat memory model to minimize segmentation. Describes two-level, three-level (PAE), and four-level paging. Linux's architecture-independent four-level page table model (PGD -> PUD -> PMD -> PTE) folds unused levels on hardware with fewer. Covers [memory zones](../entities/memory-management.md) (ZONE_DMA, ZONE_NORMAL, ZONE_HIGHMEM), TLB flushing, fix-mapped addresses, and the kernel's linear mapping of the first 896 MB at PAGE_OFFSET. See [concept-memory-addressing](../concepts/concept-memory-addressing.md).

### Ch 3: Processes
Covers the [task_struct](../entities/process-management.md) process descriptor, [thread_info](../entities/process-management.md) at the base of the 8 KB kernel stack, process states, PID allocation via pidmap_array, and the pidhash tables. Process creation via `clone()`/`fork()`/`vfork()` -> `do_fork()` -> `copy_process()` with [concept-copy-on-write](../concepts/concept-copy-on-write.md) for memory. Context switching via `switch_to`/`__switch_to()` with lazy FPU save/restore. Kernel threads (process 0 = swapper/idle, process 1 = init). Zombie lifecycle and wait queues. See [process-management](../entities/process-management.md).

### Ch 4: Interrupts and Exceptions
Covers the x86 IDT (256 entries), interrupt gates vs. trap gates, 8259A PIC and APIC systems (I/O APIC + Local APIC for SMP). The `do_IRQ()` -> `__do_IRQ()` path using `irq_desc[]` array and `irqaction` handler chains. Three deferred-work mechanisms: softirqs (6 fixed types), tasklets (dynamic, serialized per-tasklet), and work queues (process context, can sleep). Per-CPU `ksoftirqd` threads throttle softirq storms. See [interrupt-handling](../entities/interrupt-handling.md).

### Ch 5: Kernel Synchronization
Full hierarchy of primitives: per-CPU variables (no sync needed), atomic operations (`atomic_t`), memory barriers (`mb()`/`rmb()`/`wmb()`), spinlocks (`spin_lock*` variants for SMP/IRQ/BH protection), read-write spinlocks, seqlocks (optimistic read with sequence counter), RCU (lock-free reads, deferred freeing after grace period), semaphores (sleeping locks), read-write semaphores, completions, and the legacy Big Kernel Lock. Covers kernel preemption via `preempt_count`. See [concept-kernel-synchronization](../concepts/concept-kernel-synchronization.md).

### Ch 6: Timing Measurements
Hardware timers: RTC, TSC, PIT (8254), Local APIC timer, HPET, ACPI PMT. `jiffies` counter (HZ=1000 on x86). Timer interrupt path: `timer_interrupt()` -> `do_timer()` + `update_process_times()`. Wall-clock time in `xtime` (seqlock-protected). Dynamic timer wheel: hierarchical 5-level structure (`tv1`-`tv5`) providing O(1) insertion and amortized O(1) expiration. Delay functions (`udelay`), `schedule_timeout()`, and POSIX clocks/timers. See [timing-subsystem](../entities/timing-subsystem.md).

### Ch 7: Process Scheduling
The Linux 2.6 O(1) scheduler. Two priority arrays (active/expired) per CPU, each with 140 priority levels and a bitmap for O(1) highest-priority lookup. Dynamic priority = static priority (nice) + sleep-based bonus (-5 to +5). Interactive processes detected via `sleep_avg` heuristic. Active/expired array pointer swap when active empties. SCHED_FIFO and SCHED_RR real-time policies. SMP load balancing via scheduling domains (`sched_domain`/`sched_group`) modeling CPU topology. See [process-scheduler](../entities/process-scheduler.md).

### Ch 8: Memory Management
Three-layer physical memory allocator. **Buddy system**: power-of-2 page frame allocator with coalescing. **Zones**: ZONE_DMA/ZONE_NORMAL/ZONE_HIGHMEM with watermarks (`pages_min/low/high`). Per-CPU hot/cold page caches. **Slab allocator**: object-caching layer with per-cache slabs, per-CPU `array_cache` for lockless fast path, slab coloring for cache alignment. `kmalloc()`/`kfree()` use size-class slab caches. Memory pools (`mempool_t`) for guaranteed allocation. `vmalloc()` for virtually contiguous, physically discontiguous kernel allocations. See [memory-management](../entities/memory-management.md).

### Ch 9: Process Address Space
`mm_struct` memory descriptor with VMA red-black tree and linked list. `vm_area_struct` for contiguous regions with uniform permissions. Page fault handler: `do_page_fault()` -> `handle_mm_fault()` -> `handle_pte_fault()` dispatching to `do_no_page()` (demand paging), `do_wp_page()` ([concept-copy-on-write](../concepts/concept-copy-on-write.md)), or `do_swap_page()`. Anonymous zero-page optimization for read faults. Stack auto-expansion via VM_GROWSDOWN. Heap management via `brk()`/`do_brk()`. See [process-address-space](../entities/process-address-space.md).

### Ch 10: System Calls
POSIX API vs. system call distinction. x86 entry via `int $0x80` or `sysenter` -> `system_call()` entry point. System call number in `eax`, up to 6 params in registers. `sys_call_table` dispatch. User pointer verification via `access_ok()` + lazy fault handling. `copy_from_user()`/`copy_to_user()`. Signal checking and rescheduling on exit path. System call restart mechanism (`ERESTARTSYS` etc.). See [system-calls](../entities/system-calls.md).

### Ch 11: Signals
Regular signals (1-31, not queued, bitmask) vs. real-time signals (32-64, queued with `sigqueue`). Generation (`send_sig_info()`), delivery (`do_signal()` on return to user mode), and handler stack frame setup (`setup_frame()`/`setup_rt_frame()`). Signal return via `sigreturn()` system call. `SA_RESTART` for automatic syscall restart. Job control signals (SIGSTOP/SIGCONT). Thread group signal delivery. See [signals](../entities/signals.md).

### Ch 12: The Virtual Filesystem
Four core VFS objects: superblock (`super_block`), inode (`inode`), dentry (`dentry`), file (`file`), each with operations structs for dispatch. Dentry cache (dcache) for fast pathname-to-inode lookup with negative dentry caching. Pathname resolution via `path_lookup()` -> `link_path_walk()`, handling `.`, `..`, mount traversal, symlinks (depth limits: 5 nested, 40 total). Filesystem registration (`register_filesystem()`) and mounting (`vfsmount`). File locking: FL_FLOCK (per-file-object) vs. FL_POSIX (per-process-per-inode, byte-range). See [virtual-filesystem](../entities/virtual-filesystem.md).

### Ch 13: I/O Architecture and Device Drivers
I/O ports vs. memory-mapped I/O. Hierarchical bus architecture (PCI as backbone). Linux device model: `kobject` -> `kset` -> `subsystem` hierarchy exported via sysfs. `bus_type`/`device`/`device_driver` core abstractions. Resource management (`resource` tree). Character and block device registration. DMA: coherent (`dma_alloc_coherent`) and streaming (`dma_map_single`) mappings. See [device-driver-model](../entities/device-driver-model.md).

### Ch 14: Block Device Drivers
`bio` as the fundamental block I/O unit (scatter-gather segment list). `request` aggregates bios for contiguous sectors. `request_queue` per device with plugging/unplugging for batched dispatch. Four I/O schedulers: Noop (FIFO), Deadline (sorted + FIFO with expiry), Anticipatory (Deadline + brief pause for locality), CFQ (per-process queues for fairness). `gendisk` represents disks; `hd_struct` represents partitions. See [block-layer](../entities/block-layer.md).

### Ch 15: The Page Cache
`address_space` with radix tree indexing for O(1) page lookup by file offset. Subsumes the old buffer cache -- `buffer_head` structures attached to pages for per-block tracking. Read path: `find_get_page()` -> cache miss -> `readpage()`. Write path: `prepare_write()` -> copy data -> `commit_write()` -> mark dirty. Read-ahead with adaptive window sizing. Writeback by pdflush threads triggered by dirty ratio thresholds. See [page-cache](../entities/page-cache.md).

### Ch 16: Accessing Files
Read path via `generic_file_read()` -> `do_generic_file_read()`. Write path via `generic_file_write()`. Memory-mapped file I/O via `mmap()` + page fault -> `nopage` method. Direct I/O bypasses page cache. Dirty page throttling via `balance_dirty_pages()`.

### Ch 17: Page Frame Reclaiming
PFRA classifies pages: unreclaimable, swappable, syncable, discardable. Two LRU lists per zone (active/inactive). `shrink_zone()` -> `shrink_cache()` -> `shrink_list()`. Reverse mapping: `anon_vma` for anonymous pages, `i_mmap` tree for file-mapped. Swap subsystem: `swap_info_struct`, swap cache (`swapper_space`), swap token anti-thrashing. `kswapd` background reclamation. OOM killer via `badness()` scoring. See [page-frame-reclaiming](../entities/page-frame-reclaiming.md).

### Ch 18: Ext2 and Ext3 Filesystems
Ext2: block groups with bitmaps, 12 direct + 3 indirect block pointers, block preallocation. Ext3: adds JBD journaling layer with three modes -- `journal` (data+metadata), `ordered` (metadata, data-first, default), `writeback` (metadata only). JBD transaction lifecycle: running -> locked -> flush -> commit -> finished -> checkpoint. See [ext2-ext3](../entities/ext2-ext3.md).

### Ch 19: Process Communication
Pipes: circular buffer via `pipe_inode_info`, 16 page-sized buffers. FIFOs: named pipes with filesystem entries. System V IPC: semaphores (`sem_array` with undo), message queues (`msg_queue` with typed messages), shared memory (`shmid_kernel` via `shmat()`/`do_mmap()`). IPC namespaces for container isolation. POSIX message queues with priorities. See [ipc](../entities/ipc.md).

### Ch 20: Program Execution
`execve()` -> `do_execve()` -> `search_binary_handler()` iterating `linux_binfmt` list. ELF loader (`load_elf_binary()`): maps PT_LOAD segments via `do_mmap()`, sets up stack with argv/envp/auxiliary vector, maps dynamic linker for shared libraries. Script handling via `#!` interpreter. `start_thread()` sets user-mode eip/esp. Address space fully replaced on success. See [program-execution](../entities/program-execution.md).

### Appendix A: System Startup
Boot sequence: BIOS POST -> boot loader (LILO/GRUB) -> `setup()` (Real Mode -> Protected Mode) -> `startup_32()` (decompress + provisional page tables + enable paging) -> `start_kernel()` (initializes all subsystems) -> process 1 runs `/sbin/init`. See [boot-process](../entities/boot-process.md).

### Appendix B: Modules
Loadable kernel modules via `module` structure. Loading: `insmod` -> `sys_init_module()` (ELF validation, symbol relocation, `init()` call). Unloading: `rmmod` -> `sys_delete_module()` (ref count check, `exit()` call). Symbol export via `EXPORT_SYMBOL`/`EXPORT_SYMBOL_GPL`. Dependency tracking via `modules_which_use_me`. Demand loading via `request_module()` -> `modprobe`. See [kernel-modules](../entities/kernel-modules.md).

## Key Themes

1. **Bottom-up architecture**: The book starts with hardware mechanisms and builds up to portable abstractions, reflecting how the kernel itself is structured.
2. **Performance through caching**: Multiple cache layers (TLB, dentry cache, page cache, slab cache, per-CPU caches) are central to kernel performance.
3. **Concurrency management**: From per-CPU variables to RCU, the kernel provides a rich hierarchy of synchronization primitives tuned to different access patterns.
4. **Demand-driven allocation**: Memory is allocated lazily via demand paging, COW, and zero-page optimization, minimizing unnecessary work.
5. **Pluggable policies**: I/O schedulers, filesystem types, binary formats, and clock sources are all pluggable via function-pointer dispatch tables.

## See also

- [process-management](../entities/process-management.md)
- [memory-management](../entities/memory-management.md)
- [virtual-filesystem](../entities/virtual-filesystem.md)
- [process-scheduler](../entities/process-scheduler.md)
- [concept-kernel-synchronization](../concepts/concept-kernel-synchronization.md)
- [interrupt-handling](../entities/interrupt-handling.md)
