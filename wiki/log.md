---
type: log
created: 2026-04-08
---

# Wiki Log

Chronological record of wiki activity.

## [2026-04-08] init | Wiki created

Initialized the kernel-wiki with directory structure, schema (CLAUDE.md), index, log, and overview page. Ready for source ingestion.

## [2026-04-08] ingest | Understanding the Linux Kernel, 3rd Edition

Ingested the full book (20 chapters + 2 appendices, ~940 pages) by Daniel P. Bovet and Marco Cesati. Covers Linux kernel 2.6.11 on x86 architecture. Created the following pages:

**Source summary:**
- [src-understanding-the-linux-kernel](sources/src-understanding-the-linux-kernel.md) — Chapter-by-chapter summary with key themes

**Entity pages (17):**
- [process-management](entities/process-management.md) — task_struct, fork/clone, context switching, kernel threads, wait queues, PID management
- [process-scheduler](entities/process-scheduler.md) — O(1) scheduler, priority arrays, dynamic priority, SMP load balancing, scheduling domains
- [system-calls](entities/system-calls.md) — int $0x80/sysenter entry, sys_call_table, parameter verification, restart mechanism
- [signals](entities/signals.md) — Regular vs real-time signals, generation/delivery/handler setup, job control
- [program-execution](entities/program-execution.md) — execve, ELF loading, dynamic linker, script handling, address space replacement
- [memory-management](entities/memory-management.md) — Buddy system, zones, slab allocator, kmalloc/kfree, vmalloc, memory pools
- [process-address-space](entities/process-address-space.md) — mm_struct, VMAs, page fault handler, demand paging, COW, heap via brk
- [page-cache](entities/page-cache.md) — address_space/radix tree, buffer_head integration, read-ahead, pdflush writeback
- [page-frame-reclaiming](entities/page-frame-reclaiming.md) — PFRA, active/inactive LRU, kswapd, reverse mapping, swap, OOM killer
- [virtual-filesystem](entities/virtual-filesystem.md) — VFS objects (superblock/inode/dentry/file), dcache, pathname lookup, file locking
- [ext2-ext3](entities/ext2-ext3.md) — Block groups, indirect block addressing, JBD journaling (journal/ordered/writeback modes)
- [block-layer](entities/block-layer.md) — bio, request queues, I/O schedulers (Noop/Deadline/Anticipatory/CFQ), queue plugging
- [device-driver-model](entities/device-driver-model.md) — kobject/sysfs, bus/device/driver model, PCI subsystem, I/O ports, DMA
- [interrupt-handling](entities/interrupt-handling.md) — IDT, PIC/APIC, do_IRQ, softirqs, tasklets, work queues, ksoftirqd
- [timing-subsystem](entities/timing-subsystem.md) — Hardware timers, jiffies, timer wheel, xtime, udelay, POSIX clocks, NTP
- [ipc](entities/ipc.md) — Pipes, FIFOs, System V IPC (semaphores/messages/shared memory), POSIX message queues
- [boot-process](entities/boot-process.md) — BIOS -> bootloader -> setup -> startup_32 -> start_kernel -> init
- [kernel-modules](entities/kernel-modules.md) — module struct, insmod/rmmod, EXPORT_SYMBOL, demand loading, taint flags

**Concept pages (3):**
- [concept-kernel-synchronization](concepts/concept-kernel-synchronization.md) — Full hierarchy: per-CPU vars, atomics, barriers, spinlocks, seqlocks, RCU, semaphores, completions, BKL
- [concept-copy-on-write](concepts/concept-copy-on-write.md) — COW in fork, do_wp_page, anonymous zero-page optimization
- [concept-memory-addressing](concepts/concept-memory-addressing.md) — x86 address translation, flat model, four-level paging, TLB, kernel memory layout

## [2026-04-08] restructure | Wiki directory reorganization

Reorganized wiki into subdirectories by page type per gist.md architecture guidelines:
- `sources/` — source summary pages
- `entities/` — entity pages (subsystems, data structures, functions)
- `concepts/` — concept pages (cross-cutting ideas)
- `comparisons/` — comparison pages (empty, ready for use)
- `analyses/` — analysis pages (empty, ready for use)

Root-level files (`index.md`, `log.md`, `overview.md`) remain at `wiki/` top level. Updated all wikilinks across 25 files to use subdirectory paths. Updated CLAUDE.md schema accordingly.

## [2026-04-08] lint | Full health check and fixes

Scanned all 24 wiki pages. Found and fixed:
- **5 broken pipe-alias wikilinks** — `[[page|alias]]` format missed during directory restructure; added subdirectory prefixes
- **1 false wikilink** in overview.md — `[[wikilinks]]` was a syntax example, not a real link
- **29 broken links to non-existent pages** — redirected to existing relevant pages using pipe aliases (e.g., `[[buddy-system]]` → `[[entities/memory-management|buddy system]]`)
- **2 self-links** removed from See also sections (process-management, process-scheduler)
- **3 over-long pages** trimmed to under 300 lines by condensing verbose sections (device-driver-model 304→288, virtual-filesystem 316→299, concept-kernel-synchronization 306→299)

Post-fix verification: 0 broken links, 0 orphan pages, 0 over-long pages, index complete.

## [2026-04-09] ingest | Advanced Linux Programming

Ingested the full book (11 chapters + 6 appendixes, ~340 pages) by Mark Mitchell, Jeffrey Oldham, and Alex Samuel. A practical guide to GNU/Linux application development — covers the same kernel features as the internals books but from the **user-space programmer's perspective**: how to use fork/exec, pthreads, IPC mechanisms, signals, /proc, devices, security features, and system calls correctly.

**Source summary:**
- [src-advanced-linux-programming](sources/src-advanced-linux-programming.md) — Chapter-by-chapter summary with key themes

**New entity pages (4):**
- [pthreads](entities/pthreads.md) — POSIX threads: creation/joining, cancellation, thread-specific data, mutexes, semaphores, condition variables, thread vs. process trade-offs
- [proc-filesystem](entities/proc-filesystem.md) — /proc virtual filesystem: per-process entries (/proc/\<pid\>/status, maps, fd/), hardware info, kernel params, /proc/sys/ tunables
- [devices](entities/devices.md) — Device types (character/block), device nodes in /dev, special devices (/dev/null, /dev/random, etc.), ioctl interface, pseudo-terminals
- [gnu-toolchain](entities/gnu-toolchain.md) — GCC compilation pipeline, GNU Make, GDB, strace/ltrace, valgrind, gprof, gcov, objdump

**New concept pages (2):**
- [concept-linux-security](concepts/concept-linux-security.md) — Users/groups, process credentials (real/effective/saved UIDs), set-UID programs, file permissions, POSIX capabilities, chroot jails, PAM
- [concept-inline-assembly](concepts/concept-inline-assembly.md) — GCC inline asm (GAS/AT&T syntax), extended asm with constraints, volatile, common patterns (rdtsc, cpuid, memory barriers)

**Updated entity pages (4):**
- [ipc](entities/ipc.md) — Added memory-mapped file IPC, UNIX domain sockets (stream/datagram, ancillary data, FD passing), IPC mechanism comparison table
- [signals](entities/signals.md) — Added user-space API section: signal() vs. sigaction(), common patterns (child reaping, graceful shutdown, double-fork), async-signal safety
- [system-calls](entities/system-calls.md) — Added user-space system call reference (access, fcntl, fsync, mmap, sendfile, setitimer, etc.) and strace usage
- [process-management](entities/process-management.md) — Added user-space process programming: exec family table, zombie prevention strategies, process termination (exit vs. _exit vs. abort)

## [2026-04-08] ingest | Linux Kernel in a Nutshell

Ingested the full book (11 chapters + 2 appendices, ~184 pages) by Greg Kroah-Hartman. A practical guide to building, configuring, and installing the Linux 2.6 kernel. Complements the theoretical coverage from "Understanding the Linux Kernel."

**Source summary:**
- [src-linux-kernel-in-a-nutshell](sources/src-linux-kernel-in-a-nutshell.md) — Chapter-by-chapter summary with key themes

**New entity pages (2):**
- [kernel-build-system](entities/kernel-build-system.md) — Kbuild: make targets, parallel/cross-compilation, out-of-tree builds, ccache/distcc, module installation, cleaning/packaging targets
- [kernel-configuration](entities/kernel-configuration.md) — Kconfig: menuconfig/xconfig/gconfig/oldconfig/defconfig, .config file format, tristate options, device discovery workflow, comprehensive option reference by category

**Updated entity pages (2):**
- [boot-process](entities/boot-process.md) — Added kernel boot parameters section (console, root, init, mem, debug, CPU, timer, scheduler parameters)
- [kernel-modules](entities/kernel-modules.md) — Added module installation (`make modules_install`, `/lib/modules/<version>/` layout, `modules.dep`/`modules.alias`) and boot-time module parameter passing

## [2026-04-09] ingest | Understanding Linux Network Internals (complete: all 36 chapters, 1064 pages)

Ingested the complete book (1064 pages, 36 chapters in 7 parts) by Christian Benvenuti (O'Reilly, 2005). Comprehensive coverage of the Linux 2.6 networking stack internals — from device drivers and frame reception/transmission through L2 bridging, IPv4, the neighboring subsystem (ARP), and routing. All pages read across 6 parallel reader agents.

**Source summary:**
- [src-understanding-linux-network-internals](sources/src-understanding-linux-network-internals.md) — Part-by-part summary covering sk_buff, net_device, device init, frame TX/RX, bridging, IPv4, ARP/neighboring, and routing

**New entity pages (12):**
- [sk-buff](entities/sk-buff.md) — Socket buffer: packet representation, buffer pointers (head/data/tail/end), protocol headers, cloning, queue operations
- [net-device](entities/net-device.md) — Network device: interface representation, function pointers (open/stop/hard_start_xmit), registration lifecycle, state machine, virtual devices
- [network-device-initialization](entities/network-device-initialization.md) — Notification chains (netdev_chain, inetaddr_chain), PCI NIC probing, net_dev_init(), alloc_etherdev, device registration/open/close
- [frame-reception](entities/frame-reception.md) — NAPI and legacy (netif_rx) reception, softnet_data, net_rx_action budget, protocol demultiplexing via netif_receive_skb
- [frame-transmission](entities/frame-transmission.md) — dev_queue_xmit, hard_start_xmit, TX flow control (netif_stop/wake_queue), NET_TX_SOFTIRQ, watchdog timer
- [bridging](entities/bridging.md) — L2 bridging: net_bridge/net_bridge_port, STP (802.1D port states, BPDUs, topology change), forwarding database (MAC learning, aging), brctl
- [ipv4-subsystem](entities/ipv4-subsystem.md) — IPv4: ip_rcv/ip_forward/ip_local_deliver pipeline, ip_queue_xmit/cork transmission, ip_fragment/ip_defrag, ICMPv4, inet_protos[] L4 dispatch, inet_peer
- [neighboring-subsystem](entities/neighboring-subsystem.md) — Protocol-independent neighbor infrastructure: neigh_table, neighbour struct, NUD state machine, L2 header caching, garbage collection
- [arp](entities/arp.md) — ARP: arphdr packet format, arp_process() logic, gratuitous ARP, proxy ARP, arp_filter/arp_announce/arp_ignore, ARPD, RARP
- [routing-subsystem](entities/routing-subsystem.md) — Routing architecture: two-level cache+FIB design, rtable/dst_entry/flowi/fib_result structures, input/output routing functions, initialization, external events
- [routing-cache](entities/routing-cache.md) — Routing cache: rt_hash_table hash organization, cache lookup/creation/insertion, multipath caching, synchronous+asynchronous GC, cache flush, ICMP redirect handling
- [routing-tables-fib](entities/routing-tables-fib.md) — FIB: fib_table/fib_info/fib_nh/fib_node/fib_alias structures, fn_hash (hash per prefix length) and fn_trie (LC-trie) backends, route add/delete flows, policy routing rules

**New concept pages (3):**
- [concept-napi-interrupt-mitigation](concepts/concept-napi-interrupt-mitigation.md) — NAPI: interrupt+polling hybrid, budget mechanism, livelock prevention, comparison with legacy reception
- [concept-policy-routing](concepts/concept-policy-routing.md) — Multiple routing tables with rule-based selection, fib_rule matching (src/dst/tos/fwmark), actions, RTN_THROW, practical examples
- [concept-network-packet-flow](concepts/concept-network-packet-flow.md) — End-to-end packet flow: ingress (NIC interrupt -> NAPI -> protocol demux -> routing -> local delivery/forwarding) and egress (socket -> IP -> routing -> neighbor -> device) paths, netfilter hook points

## [2026-04-09] ingest | Professional Linux Kernel Architecture

Ingested the complete book (19 chapters + 6 appendices, ~1370 pages) by Wolfgang Mauerer (Wrox, 2008). Covers Linux kernel 2.6.24 with multi-architecture (x86, x86-64, ARM, MIPS) perspective. Read across 9 parallel reader agents (≤150 pages each). This book documents several major architectural changes from the 2.6.11 kernel covered by Bovet & Cesati.

**Source summary:**
- [src-professional-linux-kernel-architecture](sources/src-professional-linux-kernel-architecture.md) — Chapter-by-chapter summary with key themes and relationship to other sources

**New entity pages (8):**
- [cfs-scheduler](entities/cfs-scheduler.md) — Completely Fair Scheduler: vruntime, red-black tree, sched_entity/cfs_rq, group scheduling, sleeper fairness, granularity controls
- [namespaces](entities/namespaces.md) — Linux namespaces: nsproxy, PID/UTS/IPC/mount/network/user namespaces, CLONE_NEW* flags, container building blocks
- [tcp-udp](entities/tcp-udp.md) — TCP/UDP transport layer: tcp_sock, congestion control (slow start, CUBIC), three-way handshake, udp_sock, pluggable congestion algorithms
- [socket-layer](entities/socket-layer.md) — Socket layer: struct socket/sock duality, proto_ops/proto vtables, inet_create() flow, send/receive data paths
- [netfilter](entities/netfilter.md) — Netfilter framework: 5 hook points, nf_hook_ops, iptables tables/chains, connection tracking (nf_conntrack), NAT
- [extended-attributes-acls](entities/extended-attributes-acls.md) — Extended attributes (4 namespaces, xattr_handler) and POSIX ACLs (posix_acl, permission algorithm, ACL_MASK)
- [auditing](entities/auditing.md) — Audit subsystem: audit_context, NETLINK_AUDIT communication, audit rules/filters, SELinux AVC integration
- [usb-subsystem](entities/usb-subsystem.md) — USB: usb_device/usb_interface model, URBs, usb_driver matching, host controller drivers, device enumeration

**New concept pages (2):**
- [concept-scheduling-classes](concepts/concept-scheduling-classes.md) — Pluggable scheduling class framework: sched_class vtable, rt→fair→idle priority chain, dispatch mechanism, design pattern analysis
- [concept-page-mobility](concepts/concept-page-mobility.md) — Page mobility/anti-fragmentation: MIGRATE_UNMOVABLE/RECLAIMABLE/MOVABLE types, per-type buddy free lists, ZONE_MOVABLE, pageblock grouping

**Updated entity pages (5):**
- [process-scheduler](entities/process-scheduler.md) — Added CFS section with vruntime model, scheduling classes overview, cross-references to new pages
- [memory-management](entities/memory-management.md) — Added SLUB allocator, page mobility/anti-fragmentation, NUMA enhancements sections
- [interrupt-handling](entities/interrupt-handling.md) — Added genirq framework section (irq_chip, flow handlers, IRQ affinity)
- [page-frame-reclaiming](entities/page-frame-reclaiming.md) — Added lumpy reclaim and swap token enhancements sections
- [virtual-filesystem](entities/virtual-filesystem.md) — Added VFS evolution section (write_begin/write_end, struct path, RCU dcache, read_inode deprecation)

## [2026-04-09] ingest | QEMU/KVM Source Code Analysis and Application

Ingested the full book (8 chapters, ~1115 pages) by Li Qiang (李强), China Machine Press, 2021. A source-code-level analysis of the QEMU/KVM virtualization stack covering QEMU 2.8.1, Linux kernel 4.4.161, and SeaBIOS rel-1.11.2. Read across 8 parallel reader agents (≤150 pages each). This is the first source covering **virtualization** — a topic not addressed by the other five books.

**Source summary:**
- [src-qemu-kvm-source-code-and-application](sources/src-qemu-kvm-source-code-and-application.md) — Chapter-by-chapter summary with key themes and relationship to other sources

**New entity pages (8):**
- [qemu-kvm-overview](entities/qemu-kvm-overview.md) — QEMU/KVM architecture, QOM type system (TypeInfo/TypeImpl/ObjectClass/Object), QDev device model, GLib event loop, BQL, HMP/QMP monitor, QAPI
- [kvm-cpu-virtualization](entities/kvm-cpu-virtualization.md) — Intel VT-x (VMX root/non-root), VMCS, VCPU lifecycle (QEMU + KVM sides), kvm_cpu_exec loop, KVM_RUN ioctl, kvm_run shared structure
- [kvm-memory-virtualization](entities/kvm-memory-virtualization.md) — Address translation layers (GVA→GPA→HVA→HPA), QEMU MemoryRegion tree, KVM memory slots, EPT/shadow page tables, MMIO dispatch, dirty page tracking (write-protect + PML)
- [kvm-interrupt-virtualization](entities/kvm-interrupt-virtualization.md) — PIC (8259A), I/O APIC, Local APIC emulation, GSI routing, MSI, APICv (posted-interrupt descriptor, virtual-APIC page), IRQfd, IOeventfd
- [virtio-framework](entities/virtio-framework.md) — VRing (descriptor/available/used rings), VirtQueue, VirtIODevice, PCI transport, virtio-net (TX/RX, mergeable buffers, TAP, multiqueue), virtio-blk (IOThread data plane)
- [vhost](entities/vhost.md) — Vhost kernel data-plane offloading: vhost_dev/vhost_net structures, vhost_worker kernel thread, use_mm trick, ioctl interface, ioeventfd/irqfd notification bypass
- [vm-live-migration](entities/vm-live-migration.md) — migration_thread 4 phases, RAM dirty page sync (KVM_GET_DIRTY_LOG), convergence control, XBZRLE compression, post-copy via userfaultfd, VMState serialization
- [qemu-machine-emulation](entities/qemu-machine-emulation.md) — Intel 440FX/PIIX3 chipset, SeaBIOS boot, fw_cfg device, ACPI table generation, interrupt controllers, timers (PIT/HPET), VGA emulation, serial port

**New concept pages (2):**
- [concept-hardware-virtualization](concepts/concept-hardware-virtualization.md) — Hardware virtualization history, Type 1 vs Type 2 hypervisors, VT-x/VMX modes, EPT/NPT, APICv/AVIC, VT-d/IOMMU, progressive VM Exit elimination
- [concept-virtio-data-plane](concepts/concept-virtio-data-plane.md) — Data plane vs control plane, baseline QEMU → IOThread → vhost → device passthrough optimization levels, ioeventfd/irqfd notification paths

## [2026-04-09] ingest | Mastering KVM Virtualization

Ingested the full book (16 chapters, ~468 pages) by Humble Devassy Chirammal, Prasad Mukhedkar, and Anil Vettathu (Packt, 2016). An operations-focused guide to KVM virtualization covering libvirt management, networking, storage, live migration, performance tuning, V2V/P2V conversion, and OpenStack integration. Read across 4 parallel reader agents (≤150 pages each). This is the **operational counterpart** to the source-code-level QEMU/KVM analysis from Li Qiang.

**Source summary:**
- [src-mastering-kvm-virtualization](sources/src-mastering-kvm-virtualization.md) — Chapter-by-chapter summary with key themes and relationship to other sources

**New entity pages (6):**
- [libvirt-management](entities/libvirt-management.md) — libvirt architecture (libvirtd, connection URIs), virsh CLI, virt-install, virt-manager, domain XML, QMP/QEMU Monitor, monitoring/logging, libguestfs tools
- [kvm-networking](entities/kvm-networking.md) — Linux bridge, libvirt network types (NAT/routed/isolated/bridged), Open vSwitch (VLANs, GRE/VXLAN), macvtap modes, SR-IOV passthrough, nwfilter, vhost-net/multiqueue
- [kvm-storage](entities/kvm-storage.md) — Storage pools/volumes, raw vs qcow2 formats, qemu-img tool, pool types (directory/LVM/NFS/iSCSI/Ceph RBD/GlusterFS), cache modes, I/O modes, virtio-blk vs virtio-scsi
- [kvm-performance-tuning](entities/kvm-performance-tuning.md) — CPU pinning (vcpupin/emulatorpin), NUMA tuning, hugepages (2MB/1GB), KSM, I/O tuning (IOThreads, schedulers, blkdeviotune), vhost-net, tuned profiles
- [vm-snapshots-templates](entities/vm-snapshots-templates.md) — VM templates (virt-sysprep, virt-builder), cloning (virt-clone, linked clones), internal/external snapshots, blockcommit/blockpull
- [v2v-p2v-migration](entities/v2v-p2v-migration.md) — virt-v2v (VMware/Xen/Hyper-V to KVM conversion, input/output modes), virt-p2v (bootable ISO + conversion server)

**Updated entity pages (1):**
- [vm-live-migration](entities/vm-live-migration.md) — Added libvirt migration operations section: migration modes (direct/P2P/tunnelled), virsh flags, tuning, CPU compatibility, TLS security

## [2026-04-09] ingest | Debugging with GDB (Tenth Edition)

Ingested the official GNU GDB manual (33 chapters + appendices) by Richard Stallman, Roland Pesch, Stan Shebs, et al. (FSF, 2013). The definitive reference for GDB version 7.6.50, covering all debugger features from basic breakpoints through reverse debugging, tracepoints, remote debugging (RSP), the Python scripting API, and the GDB/MI machine interface.

**Source summary:**
- [src-debugging-with-gdb](sources/src-debugging-with-gdb.md) — Chapter-by-chapter summary covering all 33 chapters and appendices, key themes, and relationship to other wiki sources

**New entity pages (2):**
- [gdb-debugger](entities/gdb-debugger.md) — Core GDB reference: invocation, running programs, breakpoints/watchpoints/catchpoints, stepping, signal handling, reverse execution, tracepoints, stack/source/data examination, checkpoints, TUI, Python scripting API
- [gdb-remote-debugging](entities/gdb-remote-debugging.md) — Remote debugging: gdbserver, Remote Serial Protocol (packet format, core packets, stop replies, vCont, qSupported), agent expressions, target descriptions, .gdb_index, File-I/O extension

**New concept pages (1):**
- [concept-reverse-debugging](concepts/concept-reverse-debugging.md) — Reverse debugging and execution recording: record full vs record btrace, reverse execution commands, checkpoints, watchpoint+reverse patterns, tracepoints as non-intrusive alternative

## [2026-04-09] create | Comparison: SLAB vs SLUB vs SLOB allocators

Created a new comparison page analyzing the trade-offs between the three Linux kernel slab allocator implementations.

**New comparison page:**
- [cmp-slab-slub-slob](comparisons/cmp-slab-slub-slob.md) — Side-by-side analysis of SLAB, SLUB, and SLOB allocators covering architecture, memory efficiency, SMP/NUMA scalability, debugging, allocation latency, cache behavior, and current mainline status

**Updated:**
- [index.md](index.md) — Added comparison page entry

## [2026-04-09] create | VFIO/IOMMU Device Passthrough

Created a new entity page covering VFIO and IOMMU-based device passthrough for KVM guests. Fills the gap identified in overview.md for VFIO/device passthrough details.

**New entity page:**
- [vfio-device-passthrough](analyses/vfio-device-passthrough.md) — IOMMU hardware (VT-d/AMD-Vi), DMA remapping, interrupt remapping, IOMMU groups, VFIO kernel framework (container/group/device hierarchy), QEMU integration (DMA mapping, BAR exposure, irqfd interrupts), SR-IOV passthrough (PF/VF), security model, trade-offs

**Updated:**
- [index.md](index.md) — Added entity page entry
- [overview.md](overview.md) — Added VFIO coverage, updated gaps section
- [kvm-networking](entities/kvm-networking.md) — Added cross-link
- [concept-hardware-virtualization](concepts/concept-hardware-virtualization.md) — Added cross-link
- [concept-virtio-data-plane](concepts/concept-virtio-data-plane.md) — Added cross-link

## [2026-04-10] create | Analysis: Interrupt Delivery Process

Created a new analysis page tracing end-to-end interrupt delivery for both running and halted CPU scenarios.

**New analysis page:**
- [analysis-interrupt-delivery-process](analyses/analysis-interrupt-delivery-process.md) — Complete interrupt delivery lifecycle: I/O APIC / MSI routing, LAPIC acceptance (IRR → priority filter → INTR), running CPU scenario (IF check, instruction boundary, hardware context save, IDT dispatch, do_IRQ), halted CPU scenario (C-state exit, HLT wakeup, sti;hlt atomicity), IPI delivery for reschedule/TLB shootdown, edge vs. level trigger modes, latency comparison table, practical tuning knobs, KVM posted-interrupt crossover

**Updated:**
- [index.md](index.md) — Added first analysis page entry

## [2026-04-10] create | Chinese version of interrupt delivery analysis

Created a Chinese translation of the interrupt delivery process analysis page.

**New analysis page:**
- [analysis-interrupt-delivery-process-zh](analyses/analysis-interrupt-delivery-process-zh.md) — 中断传递流程（中文版）：完整中断传递生命周期的中文翻译，涵盖运行与停机 CPU 场景

**Updated:**
- [index.md](index.md) — Added Chinese analysis entry
- [analysis-interrupt-delivery-process](analyses/analysis-interrupt-delivery-process.md) — Added cross-link to Chinese version

## [2026-04-10] create | Chinese version of VFIO device passthrough entity

Created a Chinese translation of the VFIO/IOMMU device passthrough entity page.

**New entity page:**
- [vfio-device-passthrough-zh](analyses/vfio-device-passthrough-zh.md) — VFIO 与 IOMMU 设备直通（中文版）

**Updated:**
- [index.md](index.md) — Added Chinese entity entry
- [vfio-device-passthrough](analyses/vfio-device-passthrough.md) — Added cross-link to Chinese version

## [2026-04-10] move | vfio-device-passthrough to analyses

Moved `entities/vfio-device-passthrough.md` and `entities/vfio-device-passthrough-zh.md` to `analyses/`. Updated frontmatter type from entity to analysis. Updated all inbound links (index.md, concept-virtio-data-plane.md, concept-hardware-virtualization.md, kvm-networking.md) and internal See also links in both files.

## [2026-04-10] lint | Health check and fixes

Scanned all 80 wiki pages. Found and fixed:
- **4 broken links** in `analyses/vfio-device-passthrough.md` and `analyses/vfio-device-passthrough-zh.md` — inline body links to `vhost.md` and `kvm-networking.md` still used same-directory paths after move to `analyses/`; fixed to `../entities/` relative paths
- **8 missing `title:` fields** in all source summary pages — added `title:` to frontmatter of all 8 `sources/src-*.md` files per schema requirements

Not fixed (accepted):
- 4 over-length analysis pages (337-438 lines) — analyses are long by nature

Post-fix verification: 0 broken links, 0 orphan pages, 0 missing frontmatter fields.

## [2026-04-10] create | Comparison: Emulated PMU vs Mediated PMU

Created a new comparison page analyzing the two KVM vPMU architectures: the traditional perf-based Emulated PMU and the newer Mediated PMU with direct hardware counter access. Based on existing wiki sources and Sean Christopherson's mediated PMU patch series (`vmx/mediated_pmu_freeze_in_smm` branch, lore: `20260207041011.913471-*-seanjc@google.com`).

**New comparison page:**
- [cmp-emulated-vs-mediated-pmu](comparisons/cmp-emulated-vs-mediated-pmu.md) — Side-by-side analysis covering: core mechanism (perf events vs direct MSR load/store), VM-Exit overhead (full intercept vs MSR passthrough), counter accuracy, host-guest PMU coexistence, event filter enforcement, hardware requirements (PMU v4+, full-width writes), context switch cost, nested virtualization implications, and placement within the hardware virtualization trajectory (EPT → APICv → VT-d → Mediated PMU)

**Updated:**
- [index.md](index.md) — Added comparison page entry

## [2026-04-10] create | Chinese version of Emulated vs Mediated PMU comparison

Created a Chinese translation of the Emulated vs Mediated PMU comparison page.

**New comparison page:**
- [cmp-emulated-vs-mediated-pmu-zh](comparisons/cmp-emulated-vs-mediated-pmu-zh.md) — 模拟 PMU 与中介 PMU 对比分析（中文版）：核心机制、VM-Exit 开销、计数精度、宿主-客户共存、事件过滤、硬件要求、上下文切换成本

**Updated:**
- [index.md](index.md) — Added Chinese comparison page entry
- [cmp-emulated-vs-mediated-pmu](comparisons/cmp-emulated-vs-mediated-pmu.md) — Added cross-link to Chinese version

## [2026-04-10] create | KVM PMU Virtualization entity page

Created a comprehensive entity page covering KVM PMU virtualization architecture, data structures, and implementation details for both emulated and mediated modes. Based on Sean Christopherson's `vmx/mediated_pmu_freeze_in_smm` branch code.

**New entity page:**
- [kvm-pmu-virtualization](entities/kvm-pmu-virtualization.md) — KVM vPMU architecture: `struct kvm_pmc` (per-counter state with emulated_counter/perf_event for emulated, direct counter for mediated), `struct kvm_pmu` (per-vCPU PMU state), `struct kvm_pmu_ops` (vendor dispatch with mediated_load/put/write_global_ctrl callbacks), capability detection (`kvm_init_pmu_capability`), MSR handling (counter read/write path divergence), MSR intercept configuration (passthrough table for mediated mode), counter reprogramming (`reprogram_counter` vs `kvm_mediated_pmu_refresh_event_filter`), emulated instruction counting, context switch flows (load/put with `perf_load/put_guest_context`), PMI delivery paths, event filtering, PEBS support, module parameters

**Updated:**
- [index.md](index.md) — Added entity page under new PMU section
- [overview.md](overview.md) — Added PMU virtualization coverage, updated page counts and gaps
- [kvm-cpu-virtualization](entities/kvm-cpu-virtualization.md) — Added cross-link to PMU page
- [concept-hardware-virtualization](concepts/concept-hardware-virtualization.md) — Added Generation 5 (Mediated PMU) to VM-Exit elimination trajectory, added cross-link
- [cmp-emulated-vs-mediated-pmu](comparisons/cmp-emulated-vs-mediated-pmu.md) — Added cross-link to entity page
