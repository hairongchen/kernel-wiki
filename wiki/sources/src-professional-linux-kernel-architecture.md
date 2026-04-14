---
type: source
title: "Professional Linux Kernel Architecture"
created: 2026-04-09
updated: 2026-04-09
sources: [professional-linux-kernel-architecture]
tags: [source, linux-kernel, 2.6.24, cfs, namespaces, networking, vfs, memory-management]
---

# Professional Linux Kernel Architecture

**Author:** Wolfgang Mauerer
**Publisher:** Wrox / Wiley, 2008
**Pages:** ~1370 (19 chapters + 6 appendices)
**Kernel version:** Linux 2.6.24 (x86, x86-64, with multi-architecture coverage)

## Overview

A comprehensive, code-level walkthrough of the Linux 2.6.24 kernel. Compared to Bovet & Cesati (which covers 2.6.11), this book documents several major architectural changes: the **Completely Fair Scheduler (CFS)** replacing the O(1) scheduler, **namespace support** for container isolation, the **SLUB allocator** as an alternative to SLAB, **page mobility** for anti-fragmentation, the **genirq** framework for interrupt handling, and the **socket/transport layer** (TCP/UDP) internals. Multi-architecture coverage (x86, x86-64, ARM, MIPS, and others) distinguishes it from x86-only treatments.

## Chapter Summaries

### Chapter 1: Introduction and Overview (p. 1-34)

Kernel architecture overview: monolithic with loadable modules. Introduces the process/thread model, virtual memory layout (user/kernel split), system calls as the kernel entry point, and the distinction between preemptible and non-preemptible kernels. Covers kernel source tree organization.

### Chapter 2: Process Management and Scheduling (p. 35-132)

**Key new content for the wiki:**

- **CFS (Completely Fair Scheduler)** — Replaced the O(1) scheduler in 2.6.23. Uses a red-black tree sorted by `vruntime` (virtual runtime). Each task's `vruntime` advances at a rate inversely proportional to its weight (derived from nice value via `prio_to_weight[]` table). The leftmost node in the rb-tree is always the next task to run.
- **Scheduling classes** — `struct sched_class` with function pointers (`enqueue_task`, `dequeue_task`, `pick_next_task`, `put_prev_task`, `check_preempt_curr`). Classes form a priority chain: `rt_sched_class` > `fair_sched_class` > `idle_sched_class`. Each class manages its own runqueue data structure.
- **`struct sched_entity`** — Embedded in `task_struct`, holds `vruntime`, `load` weight, rb-tree node. Designed to support group scheduling (a `sched_entity` can represent a task group, not just a single task).
- **Real-time scheduling** — SCHED_FIFO and SCHED_RR in `rt_sched_class`. 100 RT priority levels (0-99), bitmap-based runqueue for O(1) selection.
- **Namespaces** — PID, UTS, IPC, mount, user, network namespaces. `struct nsproxy` groups active namespaces per task. PID namespace: `struct pid` contains `numbers[]` array of `struct upid` entries (one per namespace level), enabling hierarchical PID translation.
- **`clone()` flags** for namespaces: `CLONE_NEWPID`, `CLONE_NEWUTS`, `CLONE_NEWIPC`, `CLONE_NEWNS`, `CLONE_NEWUSER`, `CLONE_NEWNET`.

### Chapter 3: Memory Management (p. 133-288)

- **Page mobility / anti-fragmentation** — New in 2.6.24. Pages classified as `MIGRATE_UNMOVABLE`, `MIGRATE_RECLAIMABLE`, `MIGRATE_MOVABLE` in the buddy system's `free_area` lists. `ZONE_MOVABLE` pseudo-zone restricts allocations to movable pages. `__rmqueue_fallback()` handles cross-type stealing with a defined fallback order.
- **SLUB allocator** — Default since 2.6.22, alternative to SLAB. No per-CPU queues or slab coloring. Freelists embedded directly in freed objects. Metadata stored in `struct page` fields (`freelist`, `inuse`, `objects`). Cache merging: caches with compatible size/alignment share the same `kmem_cache`.
- **NUMA topology** — `pg_data_t` per node, zone fallback lists, node distance matrix, `zonelist` ordering (node-local vs. zone-based). Memory policy: `MPOL_BIND`, `MPOL_PREFERRED`, `MPOL_INTERLEAVE`.
- Architecture-independent four-level page table: PGD → PUD → PMD → PTE. Folding for architectures with fewer levels.

### Chapter 4: Virtual Process Memory (p. 289-346)

- `mm_struct` and VMA (`vm_area_struct`) management. VMA red-black tree for O(log n) lookup, augmented with a linked list for traversal.
- Page fault handler flow: `do_page_fault()` → `handle_mm_fault()` → `handle_pte_fault()`. Distinguishes demand paging, COW, anonymous page allocation, file-backed page-in.
- `get_unmapped_area()` for mmap placement. Top-down vs. bottom-up allocation strategies.
- Reverse mapping via `anon_vma` (anonymous pages) and `address_space` (file-backed pages) for efficient `try_to_unmap()`.

### Chapter 5: Locking and Interprocess Communication (p. 347-390)

- Atomic operations (`atomic_t`, `atomic_add`, `atomic_dec_and_test`), memory barriers (`mb()`, `rmb()`, `wmb()`).
- Spinlocks (raw and ticket-based), reader-writer spinlocks, RCU (`rcu_read_lock/unlock`, `synchronize_rcu`, `call_rcu`).
- Mutexes (`struct mutex`, replaces `struct semaphore` for mutual exclusion), completion variables.
- Per-CPU variables (`DEFINE_PER_CPU`, `get_cpu_var`), seqlocks for reader-writer scenarios with rare writes.
- System V semaphores, POSIX message queues at the kernel level.

### Chapter 6: Device Drivers (p. 391-472)

- Unified device model: `struct device`, `struct device_driver`, `struct bus_type`. Sysfs representation: `/sys/bus/`, `/sys/devices/`, `/sys/class/`.
- Character devices: `struct cdev`, `register_chrdev_region`, `file_operations`. Block devices: `struct gendisk`, `struct block_device`, request queue processing.
- PCI subsystem: `pci_driver`, resource management, DMA mapping (coherent vs. streaming). USB subsystem: `struct usb_device`, `struct usb_interface`, `struct urb` (USB Request Block), `struct usb_driver`.
- Platform devices and the device tree.
- I/O memory mapping: `ioremap()`, `readl()`/`writel()` vs. port I/O.

### Chapter 7: Modules (p. 473-518)

- `load_module()` implementation: ELF parsing, section processing, memory allocation (init + core sections), symbol resolution against kernel symbol table, relocation application.
- `EXPORT_SYMBOL()` vs. `EXPORT_SYMBOL_GPL()`. Symbol versioning via CRC checksums for ABI compatibility.
- Module reference counting: per-CPU counters for `try_module_get()`/`module_put()` scalability.
- Module parameters: `module_param()` macro, sysfs exposure under `/sys/module/<name>/parameters/`.

### Chapter 8: The Virtual Filesystem (p. 519-582)

- VFS object model: `struct super_block`, `struct inode`, `struct dentry`, `struct file`. Operations tables: `super_operations`, `inode_operations`, `file_operations`, `dentry_operations`.
- **API evolution**: `write_begin()`/`write_end()` replacing `prepare_write()`/`commit_write()` for page-based writes.
- Dentry cache: hash table with RCU-based `__d_lookup()` for lockless fast path.
- `struct path` (new) bundling `dentry` + `vfsmount` together.
- Filesystem registration: `register_filesystem()`, mounting via `do_kern_mount()`.
- `read_inode()` deprecation — inode initialization moves to filesystem-specific `iget5_locked()` patterns.

### Chapter 9: The Extended Filesystem Family (p. 583-642)

- Ext2 on-disk layout: superblock, group descriptors, block/inode bitmaps, inode table, data blocks. Direct + indirect block addressing (12 direct, single/double/triple indirect).
- Ext3 journaling via JBD: three modes (journal, ordered, writeback). Transaction lifecycle: running → committing → committed. `journal_t`, `handle_t`, `transaction_t`.
- Ext4 preview: extents (replacing indirect blocks), 48-bit block addressing, delayed allocation, multiblock allocator.

### Chapter 10: Filesystems without Persistent Storage (p. 643-706)

- **proc filesystem**: `struct proc_dir_entry`, `create_proc_entry()`, seq_file interface for iteration. Per-process entries (`/proc/<pid>/`), system-wide info (`/proc/meminfo`, `/proc/cpuinfo`), sysctl interface (`/proc/sys/`).
- **sysfs**: Kernel object representation in userspace. kobject → sysfs directory, attribute → sysfs file. `struct kobj_type` defines attribute show/store methods.
- **tmpfs / shmem**: `shmem_inode_info`, shared memory backed by swap. Uses the page cache with swap-backed pages.
- **ramfs**: Simplest filesystem — pages stay in page cache, never written back.

### Chapter 11: Extended Attributes and Access Control Lists (p. 707-732)

- **Extended attributes (xattrs)**: Four namespaces — `user`, `system`, `security`, `trusted`. `struct xattr_handler` with `get`/`set`/`list` operations per namespace. Stored in dedicated disk blocks (Ext2/Ext3 share xattr blocks via reference counting, Ext4 inline xattrs in inode).
- **POSIX ACLs**: `struct posix_acl` with `posix_acl_entry` array. Entry types: `ACL_USER_OBJ`, `ACL_USER`, `ACL_GROUP_OBJ`, `ACL_GROUP`, `ACL_MASK`, `ACL_OTHER`. Permission algorithm: match specific user/group entries first, apply `ACL_MASK` to group-class entries. Stored as `system.posix_acl_access` and `system.posix_acl_default` xattrs.

### Chapter 12: Networks (p. 733-818)

- **Socket layer**: `struct socket` (BSD interface) ↔ `struct sock` (network layer). `struct proto_ops` (socket operations per family) and `struct proto` (transport protocol operations). `inet_create()` flow: socket creation → protocol lookup → sock allocation.
- **`sk_buff` extensions**: GSO (Generic Segmentation Offload) fields, `sk_buff_head` for queue management, `skb_shared_info` for scatter-gather and frags.
- **TCP internals**: `struct tcp_sock` with congestion window (`snd_cwnd`), RTT estimation (`srtt`), retransmission timer. `tcp_v4_connect()` → three-way handshake. `tcp_sendmsg()` → `tcp_transmit_skb()` → IP layer. `tcp_v4_rcv()` reception with state machine.
- **UDP internals**: `struct udp_sock`, `udp_sendmsg()` / `udp_rcv()`. Simpler than TCP: no connection state, no congestion control.
- **Netfilter**: 5 hook points (`NF_INET_PRE_ROUTING`, `NF_INET_LOCAL_IN`, `NF_INET_FORWARD`, `NF_INET_LOCAL_OUT`, `NF_INET_POST_ROUTING`). `struct nf_hook_ops` registers callbacks. `NF_HOOK()` macro invokes chain. Connection tracking (`nf_conntrack`) and NAT. iptables as the userspace interface.
- **Network namespaces**: `struct net` encapsulates per-namespace networking state (routing tables, device lists, /proc/net entries).

### Chapter 13: System Calls (p. 819-846)

- System call entry: `int 0x80` (legacy) vs. `sysenter`/`syscall` (fast path). `sys_call_table` dispatch. Parameter passing via registers (x86: ebx, ecx, edx, esi, edi, ebp; x86-64: rdi, rsi, rdx, r10, r8, r9).
- `SYSCALL_DEFINE` macros for type-safe system call definition. `copy_from_user()`/`copy_to_user()` with fault handling.
- Restart mechanism for interrupted system calls (`-ERESTARTSYS`, `restart_block`).

### Chapter 14: Kernel Activities (p. 847-892)

- **genirq framework**: Generic IRQ subsystem separating flow handling from ISR invocation. `struct irq_desc` per IRQ line, `struct irq_chip` for hardware operations (`ack`, `mask`, `unmask`, `eoi`). Flow handlers: `handle_level_irq()`, `handle_edge_irq()`, `handle_fasteoi_irq()`.
- Softirqs (compile-time, limited to 32 entries), tasklets (dynamic, built on softirqs), work queues (`struct workqueue_struct`, `create_workqueue()`).
- **`ksoftirqd`** kernel thread: processes softirqs when load is excessive, preventing livelock.
- IRQ affinity: `/proc/irq/<N>/smp_affinity` for binding interrupts to specific CPUs.

### Chapter 15: Time Management (p. 893-948)

- Hardware clocks: PIT (8254), HPET, TSC, local APIC timer. `struct clocksource` abstraction for selecting the best available clock.
- **High-resolution timers (`hrtimer`)**: Nanosecond-precision timers using a red-black tree (replacing the O(1) timer wheel for high-res needs). `CLOCK_MONOTONIC` vs. `CLOCK_REALTIME`.
- **Tickless kernel (`CONFIG_NO_HZ`)**: Dynamic tick — suppress periodic timer interrupts when CPU is idle to save power. `tick_nohz_stop_sched_tick()`.
- Jiffies, `do_timer()`, wall clock maintenance. `struct timespec` / `struct timeval`.
- `itimer` (interval timers), POSIX timers (`timer_create`, `timer_settime`).

### Chapter 16: Page and Buffer Cache (p. 949-988)

- `struct address_space` as the cache manager per inode. Radix tree for page lookup. `struct buffer_head` for sub-page block mapping.
- `find_get_page()` / `find_or_create_page()` for cache access. Read-ahead: `page_cache_readahead()` with adaptive window sizing.
- `write_begin()`/`write_end()` write path. `mark_page_dirty()` and `writeback_inodes()` for dirty page flushing.
- `pdflush` kernel threads for periodic and memory-pressure-triggered writeback.

### Chapter 17: Data Synchronization (p. 989-1022)

- `struct backing_dev_info` for per-device writeback control (congestion tracking, dirty thresholds).
- Laptop mode: delay writeback to reduce disk spin-ups.
- `sync()`, `fsync()`, `fdatasync()` — system calls for explicit synchronization. `O_DIRECT` for bypassing the page cache.
- The journaling block device (JBD) layer shared between Ext3 and other filesystems.

### Chapter 18: Page Reclaim and Swapping (p. 1023-1096)

- PFRA (Page Frame Reclaiming Algorithm): `shrink_zone()` → `shrink_active_list()` / `shrink_inactive_list()`. Active/inactive LRU list management.
- **Lumpy reclaim**: New in 2.6.24. `isolate_lru_pages()` in `ISOLATE_BOTH` mode pulls contiguous physical pages regardless of LRU list membership, improving allocation of higher-order pages.
- **Swap token**: Anti-thrashing mechanism. `grab_swap_token()` grants one process protection from reclaim. Token holder's pages get artificial reference boost in `page_referenced_one()`.
- `kswapd`: Per-zone watermarks (pages_min, pages_low, pages_high). Background reclaim triggered at pages_low.
- Swap subsystem: `struct swap_info_struct`, swap extent mapping, priority-based swap device selection.
- OOM killer: `select_bad_process()` → `oom_kill_process()`. Scoring based on memory usage, nice value, and `oom_adj`.

### Chapter 19: Auditing (p. 1097-1116)

- `struct audit_context` attached to `task_struct`. System call entry/exit hooks for recording audit events.
- **Netlink communication**: `NETLINK_AUDIT` socket type for userspace ↔ kernel audit messaging. `audit_receive_msg()` handles commands from `auditd`.
- Audit rules: filter lists (entry, exit, task, user) with field-based matching.
- SELinux AVC (Access Vector Cache) auditing integration.

### Appendices

- **A: Architecture Specifics** — x86/x86-64 page table formats, system call entry conventions, TLB management.
- **B: Working with the Source Code** — LXR, cscope, git basics for kernel source navigation.
- **C: Notes on C** — GCC extensions (`__attribute__`, `typeof`, statement expressions), kernel-specific C idioms (container_of, likely/unlikely, BUILD_BUG_ON).
- **D: System Startup** — Boot process: BIOS → bootloader → compressed kernel → `start_kernel()` → `rest_init()` → init process. Architecture-specific early setup.
- **E: The ELF Binary Format** — ELF header, program headers (segments), section headers, symbol/string tables, dynamic linking, core dumps.
- **F: The Kernel Development Process** — Kernel release cycle, patch submission, coding style, tools (git, quilt).

## Key Themes

1. **CFS as a paradigm shift** — The move from O(1) to CFS represents a fundamental change: from heuristic-based priority manipulation to a mathematically fair model based on weighted virtual time.
2. **Namespace isolation** — The namespace infrastructure provides the kernel-level building blocks for containers, with hierarchical PIDs, isolated IPC, and independent network stacks.
3. **Memory anti-fragmentation** — Page mobility types and `ZONE_MOVABLE` represent a pragmatic approach to the long-standing fragmentation problem without requiring a compacting GC.
4. **Pluggable abstractions** — Scheduling classes, clock sources, IRQ flow handlers, slab allocators, and FIB backends all use the same pattern: a struct of function pointers with a registration/selection mechanism.
5. **Layered networking** — Clear separation between socket layer (BSD API), transport (TCP/UDP), network (IP), and link layer, with netfilter hooks at each transition point.

## Relationship to Other Sources

- **vs. Bovet & Cesati (2.6.11)**: Mauerer covers a newer kernel (2.6.24 vs. 2.6.11). Key differences: CFS replaces O(1) scheduler, SLUB replaces SLAB as default, genirq replaces hardcoded IRQ handling, namespaces are new, VFS API evolves (write_begin/write_end).
- **vs. Benvenuti (networking)**: Mauerer adds the transport layer (TCP/UDP) and socket layer that Benvenuti's L2-L3 focused book omits. Also covers netfilter and network namespaces.
- **vs. Kroah-Hartman**: Mauerer covers device drivers with more internal detail (unified device model, USB internals), while Kroah-Hartman focuses on practical build/configure workflows.

## See also

- [src-understanding-the-linux-kernel](src-understanding-the-linux-kernel.md)
- [src-understanding-linux-network-internals](src-understanding-linux-network-internals.md)
- [process-scheduler](../entities/process-scheduler.md)
- [memory-management](../entities/memory-management.md)
- [virtual-filesystem](../entities/virtual-filesystem.md)
- [concept-scheduling-classes](../concepts/concept-scheduling-classes.md)
- [concept-page-mobility](../concepts/concept-page-mobility.md)
