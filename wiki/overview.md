---
type: overview
created: 2026-04-08
updated: 2026-04-14
sources: [understanding-the-linux-kernel, linux-kernel-in-a-nutshell, advanced-linux-programming, understanding-linux-network-internals, professional-linux-kernel-architecture, qemu-kvm-source-code-and-application, mastering-kvm-virtualization, debugging-with-gdb, lcna-co2012-sekiyama, minimizing-vmexits-pv-ipi-passthrough-timer, all-solution-vmexit, bytedance-solution-vmexit, bytedance-solution-vmexit-code, vmexit-opt-hitachi-sekiyama]
tags: [overview]
---

# Kernel Wiki Overview

This wiki is a structured knowledge base about the **Linux kernel** — its architecture, subsystems, data structures, algorithms, and design philosophy.

## Current Coverage

The wiki covers the **Linux 2.6 kernel** (x86/x86-64) and its application in **QEMU/KVM virtualization** from seven complementary perspectives plus six specialized articles. "Understanding the Linux Kernel" (Bovet & Cesati) provides deep theoretical coverage of kernel 2.6.11 internals. "Professional Linux Kernel Architecture" (Mauerer) extends this to **2.6.24** with CFS, namespaces, SLUB, TCP/UDP, netfilter, and multi-architecture coverage. "Linux Kernel in a Nutshell" (Kroah-Hartman) adds practical guidance on building, configuring, and installing the kernel. "Advanced Linux Programming" (Mitchell, Oldham, Samuel) covers the **user-space developer's perspective**. "Understanding Linux Network Internals" (Benvenuti) provides comprehensive coverage of the **networking stack** -- from device drivers through IPv4, ARP, and routing. "QEMU/KVM Source Code Analysis and Application" (Li Qiang) covers the **QEMU/KVM virtualization stack** at source-code level -- CPU, memory, interrupt, and I/O virtualization with Intel VT-x. "Mastering KVM Virtualization" (Chirammal, Mukhedkar, Vettathu) provides the **operational counterpart** -- libvirt management, networking topologies, storage backends, performance tuning, live migration, and cloud integration. **"Debugging with GDB"** (Stallman, Pesch, Shebs) provides the **definitive GDB reference** -- breakpoints, watchpoints, tracepoints, reverse debugging, remote debugging via the GDB Remote Serial Protocol, the Python scripting API, and the GDB/MI machine interface. Six specialized articles extend the virtualization coverage: Sekiyama (Hitachi, LinuxCon 2012) on **real-time KVM via CPU isolation and direct interrupt delivery**; a Chinese-language technical analysis of the **Sekiyama direct interrupt delivery scheme** with detailed pros/cons evaluation; Huaqiao & Zhou (ByteDance, ~2020) on **timer passthrough and NoExit PV IPI** for zero-VM-Exit timer and IPI paths; @惠伟 (Zhihu) on **diagnosing and comparing three exit-less timer solutions** (Tencent/Alibaba/ByteDance); Fu Qiuwei (Volcengine/ByteDance, 2024) on the **complete edge high-performance VM architecture** with interrupt non-exit, timer passthrough, VFIO bypass, IPI extreme fastpath, and dynamic kernel isolation achieving >99% VM Exit reduction; and dengqiao.joey (ByteDance, 2020) providing the **full Passthrough IPI kernel patch** with pi_desc guest exposure reducing IPI from 7K to 2K cycles. The knowledge base contains **14 source summaries**, **61 entity pages**, **14 concept pages**, and **3 comparison pages**:

### Process Management and Scheduling
- Process lifecycle: creation (`fork`/`clone`/`vfork`), execution (`execve`/ELF loading), destruction (`do_exit`/zombie reaping). User-space patterns: the `exec` family, zombie prevention (double-fork, `SIGCHLD` handler, `SA_NOCLDWAIT`)
- The O(1) scheduler (2.6.0-2.6.22) with priority arrays and interactive process detection; the **Completely Fair Scheduler** (CFS, 2.6.23+) with red-black tree, vruntime-based fairness, and pluggable scheduling classes (`sched_class` abstraction)
- **Namespaces** for container isolation: PID, UTS, IPC, mount, network, user namespaces with hierarchical PID translation
- Signal handling: kernel internals (generation, delivery, stack frames) + user-space API (`sigaction()`, async-signal safety, graceful shutdown patterns)
- POSIX threads (pthreads): 1:1 kernel mapping via NPTL, mutexes, semaphores, condition variables, cancellation, thread-specific data
- System call entry/exit mechanisms, and program execution (ELF, dynamic linking)

### Memory Management
- Physical memory: buddy system allocator, memory zones (DMA/Normal/HighMem), slab allocator (SLAB with per-CPU caches, **SLUB** as 2.6.22+ default with simplified design), **page mobility** anti-fragmentation (MIGRATE_UNMOVABLE/RECLAIMABLE/MOVABLE, ZONE_MOVABLE)
- Virtual memory: process address spaces (VMAs), demand paging, Copy-on-Write, heap management
- Caching: page cache with radix tree indexing, read-ahead, pdflush writeback
- Reclamation: PFRA with LRU lists, kswapd, reverse mapping, swap subsystem, OOM killer, **lumpy reclaim** (2.6.24) for contiguous page recovery
- x86 address translation: segmentation (flat model), four-level paging, TLB management

### Filesystems and I/O
- Virtual Filesystem (VFS): four core objects (superblock/inode/dentry/file), dentry cache, pathname resolution, file locking, VFS API evolution (write_begin/write_end, struct path, RCU dcache)
- **Extended attributes and POSIX ACLs**: four xattr namespaces, posix_acl permission algorithm, ACL_MASK, on-disk storage
- Ext2/Ext3: block group layout, indirect block addressing, JBD journaling with three modes
- Generic block layer: bio-based I/O, four pluggable I/O schedulers (Noop, Deadline, Anticipatory, CFQ)
- Device driver model: kobject/sysfs hierarchy, bus/device/driver abstractions, PCI, DMA

### Kernel Infrastructure
- Interrupt and exception handling: IDT, PIC/APIC, top/bottom half split (softirqs, tasklets, work queues), **genirq** framework (irq_chip, flow handlers)
- **Auditing subsystem**: audit_context, NETLINK_AUDIT, system call entry/exit hooks, SELinux AVC integration
- Synchronization: 11-level primitive hierarchy from per-CPU variables through RCU to semaphores
- Timing: hardware timers, jiffies, dynamic timer wheel, wall clock, NTP
- Inter-process communication: pipes, FIFOs, System V IPC, POSIX message queues, UNIX domain sockets (with FD passing), memory-mapped files. Both kernel internals and user-space API patterns
- The /proc filesystem: per-process entries, hardware/kernel info, system statistics, /proc/sys tunables
- Device model: character vs. block devices, device nodes, ioctl, pseudo-terminals, special devices
- USB subsystem: layered architecture (HCD/core/drivers), URBs, device enumeration, host controllers (UHCI/OHCI/EHCI/xHCI)
- Boot sequence: BIOS through start_kernel() to init, comprehensive boot parameter reference
- Loadable kernel modules: loading, unloading, symbol export, demand loading, installation workflow

### Security
- Linux security model: users/groups, process credentials (real/effective/saved UIDs), set-UID programs
- File permissions (rwx, sticky bit, umask), POSIX capabilities for fine-grained privilege
- chroot jails (and their limitations), PAM for pluggable authentication
- Security best practices: least privilege, input validation, TOCTOU avoidance

### Development Tools
- **GNU toolchain**: GCC (compilation pipeline, optimization, libraries), GNU Make (pattern rules, parallel builds), GDB (breakpoints, watchpoints, core dumps)
- **GDB debugger (comprehensive)**: Breakpoints (software/hardware), watchpoints (write/read/access), catchpoints (syscall/fork/exec/signal/load), tracepoints (trace/ftrace/strace with actions and trace state variables), stepping (step/next/finish/until/advance), reverse execution (`record full`/`record btrace`, reverse-continue/step/next/finish), multi-threaded debugging (scheduler-locking, all-stop vs non-stop), Python scripting API (`gdb.Value`/`gdb.Type`/`gdb.Breakpoint`/`gdb.Command`, pretty-printing, frame unwinders, events), TUI split-screen interface
- **GDB remote debugging**: gdbserver (TCP/serial/pipe, multi-process mode), Remote Serial Protocol (RSP packet format, register/memory/breakpoint packets, vCont per-thread control, qSupported capability negotiation, non-stop mode), remote stubs for bare-metal targets, agent expression bytecodes for tracepoint evaluation, target description XML, `.gdb_index` for accelerated symbol lookup
- **Diagnostic tools**: strace (system call tracing), ltrace (library call tracing), valgrind (memory errors), gprof/gcov (profiling/coverage)
- **Inline assembly**: GCC asm construct, GAS/AT&T syntax, constraints, common patterns (rdtsc, cpuid)

### Networking Stack
- **Packet representation**: `sk_buff` (socket buffer) carries packet data + metadata through all layers; `net_device` represents network interfaces with function-pointer-based operations
- **Device initialization**: PCI bus enumeration and NIC driver probing, notification chains (`netdev_chain`, `inetaddr_chain`), `net_dev_init()` initializes per-CPU `softnet_data` and registers softirqs, device registration/open/close lifecycle
- **Frame reception**: NAPI interrupt mitigation (interrupt + polling hybrid), per-CPU `softnet_data`, `net_rx_action()` softirq with budget mechanism, protocol demultiplexing via `netif_receive_skb()`, legacy `netif_rx()` backlog path
- **Frame transmission**: `dev_queue_xmit()` through queuing discipline to `hard_start_xmit()`, TX flow control (`netif_stop/wake_queue()`), `NET_TX_SOFTIRQ` for completion and queue restart, watchdog timer
- **L2 bridging**: IEEE 802.1D bridge implementation with STP (port states, BPDUs, topology change detection), forwarding database (MAC learning, aging, flooding), `br_handle_frame()` ingress hook, `brctl` configuration
- **IPv4 subsystem**: Full packet lifecycle — `ip_rcv()` reception pipeline with 5 netfilter hook points, `ip_forward()` with TTL/MTU handling, `ip_local_deliver()` with fragment reassembly and L4 dispatch via `inet_protos[]`, `ip_queue_xmit()` (TCP) and cork mechanism (UDP) for transmission, `ip_fragment()`/`ip_defrag()` with fast/slow paths, ICMPv4 with rate limiting
- **Neighboring subsystem**: Protocol-independent L3-to-L2 address resolution with NUD state machine (incomplete/reachable/stale/delay/probe/failed). ARP (IPv4) and ND (IPv6) built on common infrastructure. Features: gratuitous ARP, proxy ARP, L2 header caching, dual GC (periodic + forced)
- **Routing**: Two-level architecture -- routing cache (hash table for fast-path O(1) lookup) + FIB (Forwarding Information Base for slow-path longest-prefix-match). Two FIB backends: fn_hash (default, hash per prefix length 0-32) and fn_trie (LC-trie for large tables). Policy routing via `fib_rules` for multi-table selection based on source/TOS/fwmark. Multipath routing with weighted next-hop selection
- **Socket layer**: `struct socket` (BSD API) ↔ `struct sock` (network layer), `proto_ops` / `proto` vtables, `inet_create()` flow, send/receive data paths, buffer flow control
- **TCP/UDP transport**: `tcp_sock` with congestion window/RTT tracking, three-way handshake, pluggable congestion algorithms (`tcp_congestion_ops`). UDP with minimal `udp_sock`, `udp_sendmsg()`/`udp_rcv()`
- **Netfilter**: 5 hook points (PRE_ROUTING through POST_ROUTING), `nf_hook_ops` registration, iptables (filter/nat/mangle tables), connection tracking (`nf_conntrack`), NAT (SNAT/DNAT)
- **Packet flow**: Ingress: NIC IRQ -> NAPI -> `netif_receive_skb()` -> `ip_rcv()` -> [netfilter] -> routing decision -> `ip_local_deliver()`/`ip_forward()`. Egress: socket -> `ip_queue_xmit()` -> [netfilter] -> routing -> `dst->neighbour->output()` -> `dev_queue_xmit()` -> NIC

### Virtualization (QEMU/KVM)
- **QEMU/KVM architecture**: QEMU as userspace device emulator, KVM as kernel module (`/dev/kvm`), three execution modes (guest VMX non-root, kernel VM-Exit handler, userspace QEMU). QOM (QEMU Object Model) provides C-language polymorphism via embedded structs and vtables
- **CPU virtualization**: Intel VT-x (VMX root/non-root modes), VMCS (Virtual Machine Control Structure), VCPU lifecycle from `kvm_cpu_exec()` loop through `KVM_RUN` ioctl, VM-Exit handling, shared `kvm_run` structure
- **Memory virtualization**: GVA→GPA→HVA→HPA address translation, EPT (Extended Page Tables) for hardware-assisted two-level translation, shadow page tables for non-EPT mode, KVM memory slots, dirty page tracking via write-protection and Intel PML
- **Interrupt virtualization**: PIC (8259A), I/O APIC, Local APIC emulation with timer support, MSI bypass, APICv hardware acceleration (posted-interrupt descriptors, virtual-APIC page), IRQfd/IOeventfd for efficient event notification
- **Virtio I/O framework**: VRing shared-memory protocol (descriptor table, available ring, used ring), VirtQueue, VirtIODevice base class, PCI transport. Virtio-net (TX/RX paths, TAP backend, multiqueue), virtio-blk (IOThread data plane)
- **Vhost kernel offloading**: Data plane moved to kernel `vhost_worker` thread, `use_mm()` trick for guest memory access, ioeventfd/irqfd notification bypass eliminates QEMU from I/O hot path
- **VM live migration**: Four-phase `migration_thread`, iterative dirty page tracking with convergence control, XBZRLE compression, post-copy migration via `userfaultfd`, VMState serialization framework
- **Machine emulation**: Intel 440FX/PIIX3 chipset, SeaBIOS boot flow, fw_cfg QEMU-to-firmware communication, ACPI table generation, VGA emulation with display pipeline
- **libvirt management**: libvirtd daemon, virsh CLI, virt-install, virt-manager, domain XML configuration, connection URIs, QMP monitor, monitoring/logging, libguestfs tools
- **KVM networking**: Linux bridge with tap devices, libvirt network types (NAT/routed/isolated/bridged), Open vSwitch (VLANs, GRE/VXLAN tunnels, OpenFlow), macvtap modes (VEPA/bridge/private/passthrough), SR-IOV PCI passthrough, nwfilter firewall
- **VFIO/IOMMU device passthrough**: IOMMU hardware (VT-d/AMD-Vi), DMA remapping via second-level page tables, interrupt remapping, IOMMU groups and ACS, VFIO kernel framework (container/group/device hierarchy, vfio-pci driver binding), QEMU integration (DMA mapping, BAR exposure, irqfd interrupt delivery), SR-IOV PF/VF passthrough, security model (DMA isolation, interrupt isolation, group-level granularity)
- **KVM storage**: Storage pools and volumes abstraction, raw vs qcow2 formats (backing files, snapshots, compression, LUKS), storage backends (directory, LVM, NFS, iSCSI, Ceph RBD, GlusterFS), cache modes (none/writethrough/writeback/directsync/unsafe), virtio-blk vs virtio-scsi
- **PMU virtualization**: Two modes — emulated PMU (perf-based, counter operations via `perf_event` backend) and mediated PMU (direct hardware counter access via MSR passthrough and VMCS-based PERF_GLOBAL_CTRL switching). Detailed comparison of VM-Exit overhead, counter accuracy, host-guest coexistence, event filtering, and hardware requirements (PMU v4+)
- **Performance tuning**: CPU pinning (`vcpupin`/`emulatorpin`), NUMA topology awareness (`numatune` modes), hugepages (2MB/1GB via hugetlbfs), KSM page merging, I/O tuning (IOThreads, block I/O throttling, scheduler selection), vhost-net, multiqueue virtio-net, tuned profiles, **CPU isolation** for real-time workloads (dedicated offline CPUs for bare-metal interrupt latency)
- **Real-time and low-latency virtualization**: CPU isolation + direct interrupt delivery (Sekiyama/Hitachi — disable VMCS external interrupt exiting, NMI-based host signaling, direct EOI via x2APIC MSR passthrough), **timer passthrough** (ByteDance — direct physical LAPIC timer access, host timer offloading to preemption timer, +35.5% memcached throughput), **NoExit PVIPI** (ByteDance — pi_desc passthrough, guest-side posted-interrupt IPI, 96.4% IPI overhead reduction, full RFC patch with 7K→2K cycle improvement)
- **Exit-less timer**: Three competing vendor approaches to eliminating timer VM Exits — Tencent (posted-interrupt on other pCPU, upstream merged as `pi_inject_timer`), Alibaba (PV shared-page timer, RFC), ByteDance (direct LAPIC timer passthrough, RFC). Production diagnosis shows timer is dominant VM Exit source: 93% of external interrupts from local timer, 62.5% of MSR writes from TSC_DEADLINE
- **Edge high-performance VM** (Volcengine/ByteDance, 2024): Complete near-bare-metal virtualization framework combining interrupt non-exit mechanism, timer passthrough, VFIO interrupt bypass (direct IOMMU IRTE modification), IPI extreme fastpath (assembly-optimized), and dynamic kernel resource isolation (nohz_full/interrupt/process isolation applied dynamically on VM enter/exit). >99% VM Exit reduction, 6-16% throughput gains, deployed in production for CDN/live-streaming/acceleration
- **VM lifecycle**: Templates (`virt-sysprep`, `virt-builder`), cloning (`virt-clone`, linked clones via backing files), snapshots (internal qcow2 vs external overlay chains), blockcommit/blockpull for chain management
- **V2V/P2V migration**: virt-v2v for converting VMs from VMware/Xen/Hyper-V to KVM (driver replacement, boot config), virt-p2v for physical-to-virtual conversion via bootable ISO

### Build System and Configuration
- **Kbuild**: make targets (build, install, clean, package), parallel compilation (`-jN`), cross-compilation (`ARCH`/`CROSS_COMPILE`), out-of-tree builds, ccache/distcc acceleration
- **Kconfig**: configuration infrastructure (menuconfig/xconfig/gconfig/oldconfig), `.config` file format, tristate options (y/m/n), device discovery workflows using sysfs/lspci/lsusb
- Comprehensive Kconfig option reference covering all major subsystem categories

## Key Findings

1. **Pervasive caching** — The kernel stacks multiple cache layers (TLB, dentry cache, page cache, slab cache, per-CPU caches) to minimize expensive operations at every level.
2. **Lazy/deferred work** — From demand paging to Copy-on-Write to deferred interrupt processing (softirqs/tasklets), the kernel consistently avoids doing work until it's actually needed.
3. **Per-CPU design** — Data structures are per-CPU wherever possible (runqueues, page caches, slab caches, softirq processing) to eliminate lock contention on SMP systems.
4. **Pluggable policies** — I/O schedulers, filesystem types, binary formats, clock sources, and synchronization primitives are all swappable via function-pointer dispatch tables.
5. **Hardware abstraction with escape hatches** — The kernel provides architecture-independent abstractions (four-level page tables, generic IRQ layer) while allowing architecture-specific optimizations (lazy FPU, sysenter, APIC).
6. **Kernel/user-space symmetry** — Many kernel mechanisms have direct user-space counterparts (kernel semaphores ↔ POSIX semaphores, kernel threads ↔ pthreads, kernel spinlocks ↔ futex-based mutexes), but with different constraints and use cases. Understanding both sides reveals why the APIs are designed the way they are.
7. **Protocol-independent frameworks** — The networking stack uses protocol-independent infrastructure with protocol-specific backends. The neighboring subsystem (`neigh_table`/`neighbour`) serves both ARP (IPv4) and ND (IPv6). The FIB uses pluggable backends (`fn_hash`/`fn_trie`). The routing cache uses `dst_entry` as a protocol-independent base. This pattern enables code reuse while allowing protocol-specific optimizations.
8. **Notification-driven consistency** — Subsystems are coupled through notification chains rather than direct calls.
9. **Progressive VM Exit elimination** — Hardware virtualization evolution systematically removes hypervisor intervention from the data path: VT-x eliminates binary translation, EPT eliminates shadow page tables, APICv eliminates interrupt interception, VT-d/IOMMU eliminates device emulation. The end state approaches bare-metal performance with the hypervisor only handling management operations. A device going down sends `NETDEV_DOWN`, which triggers `fib_netdev_event()` to flush routes, which triggers `rt_cache_flush()` to invalidate the cache. This decoupled design allows adding new subscribers without modifying existing code.

## Gaps and Future Work

The following topics could benefit from dedicated wiki pages or additional sources:

- **Networking stack (remaining)** — L2 through transport layer now covered (device init, frame RX/TX, bridging, IPv4, ARP, routing, TCP/UDP, socket layer, netfilter). Still needed: traffic control (tc/qdisc), wireless, IPv6 internals
- **Security subsystem** — SELinux internals (ACL auditing covered, but policy engine/LSM framework not detailed), cgroups for resource control
- **Newer kernel versions** — Sources cover 2.6.x (up to 2.6.24); modern kernels (5.x/6.x) have further evolved: multi-queue block layer, BPF, io_uring, SLUB improvements, cgroup v2, etc. QEMU/KVM source covers kernel 4.4.161 but newer features (vhost-user, vDPA, VFIO) are not detailed
- **NUMA memory management** — Memory policies and node topology covered from Mauerer; deeper coverage of per-node reclaim and migration could be added
- **initramfs/initrd** — Covered briefly in boot process; could warrant its own page
- **Virtualization (remaining)** — QEMU/KVM stack covered comprehensively at both source-code and operational levels; VFIO/IOMMU device passthrough and PMU virtualization (emulated + mediated) now covered. Still needed: vhost-user (userspace backend), vDPA (hardware virtqueue offload), AMD SVM implementation details, nested virtualization, oVirt/OpenStack deeper coverage

## How to Use

1. **Add sources**: Drop articles, papers, book chapters, or notes into `raw/` and ask to ingest them.
2. **Ask questions**: Query the wiki on any kernel topic. Substantial answers can be filed as new pages.
3. **Browse**: Open the wiki directory in Obsidian (or any markdown viewer) and follow `[[wikilink]]`-style links between pages.
4. **Lint**: Periodically ask for a health check to find gaps, contradictions, and orphan pages.

## See also

- [index](index.md)
- [src-understanding-the-linux-kernel](sources/src-understanding-the-linux-kernel.md)
- [src-linux-kernel-in-a-nutshell](sources/src-linux-kernel-in-a-nutshell.md)
- [src-advanced-linux-programming](sources/src-advanced-linux-programming.md)
- [src-understanding-linux-network-internals](sources/src-understanding-linux-network-internals.md)
- [src-professional-linux-kernel-architecture](sources/src-professional-linux-kernel-architecture.md)
- [src-qemu-kvm-source-code-and-application](sources/src-qemu-kvm-source-code-and-application.md)
- [src-mastering-kvm-virtualization](sources/src-mastering-kvm-virtualization.md)
- [src-debugging-with-gdb](sources/src-debugging-with-gdb.md)
- [src-lcna-co2012-sekiyama](sources/src-lcna-co2012-sekiyama.md)
- [src-minimizing-vmexits-pv-ipi-passthrough-timer](sources/src-minimizing-vmexits-pv-ipi-passthrough-timer.md)
- [src-all-solution-vmexit](sources/src-all-solution-vmexit.md)
- [src-bytedance-solution-vmexit](sources/src-bytedance-solution-vmexit.md)
- [src-bytedance-solution-vmexit-code](sources/src-bytedance-solution-vmexit-code.md)
- [src-vmexit-opt-hitachi-sekiyama](sources/src-vmexit-opt-hitachi-sekiyama.md)
