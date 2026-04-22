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
- [vfio-device-passthrough](analyses/analysis-vfio-device-passthrough.md) — IOMMU hardware (VT-d/AMD-Vi), DMA remapping, interrupt remapping, IOMMU groups, VFIO kernel framework (container/group/device hierarchy), QEMU integration (DMA mapping, BAR exposure, irqfd interrupts), SR-IOV passthrough (PF/VF), security model, trade-offs

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
- [vfio-device-passthrough-zh](analyses/analysis-vfio-device-passthrough-zh.md) — VFIO 与 IOMMU 设备直通（中文版）

**Updated:**
- [index.md](index.md) — Added Chinese entity entry
- [vfio-device-passthrough](analyses/analysis-vfio-device-passthrough.md) — Added cross-link to Chinese version

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

## [2026-04-10] create | Analysis: VM Exit Reduction and Timer Virtualization

Created a comprehensive analysis page (in Chinese) covering the history of VM Exit elimination techniques and timer virtualization optimization in KVM. Synthesizes information from existing wiki pages and additional domain knowledge.

**New analysis page:**
- [analysis-vm-exit-reduction-and-timer-virtualization](analyses/analysis-vm-exit-reduction-and-timer-virtualization.md) — VM Exit 消除发展历程与 Timer 虚拟化优化：五代硬件演进（VT-x → EPT → APICv → VT-d → Mediated PMU），Timer VM Exit 三大来源（LAPIC Timer/PIT/HPET），六阶段优化技术（Tickless → TSC-deadline → APICv → Preemption Timer → MSR bitmap → kvmclock），未来展望（机密计算、极致低延迟）

**Updated:**
- [index.md](index.md) — Added analysis page entry

## [2026-04-10] update | Enhanced Timer VM Exit analysis with kernel source details

Enhanced the VM Exit reduction and timer virtualization analysis page with detailed source-level information from the latest Linux kernel (`arch/x86/kvm/lapic.c`, `arch/x86/kvm/vmx/vmx.c`, `arch/x86/kvm/lapic.h`). Major additions:

- **`kvm_timer` struct** definition and field-by-field explanation
- **VMX Preemption Timer (hv_timer)** deep-dive: `kvm_can_use_hv_timer()` decision logic, `vmx_set_hv_timer()` TSC delta calculation, fastpath handling (`handle_fastpath_preemption_timer()` returning `EXIT_FASTPATH_REENTER_GUEST` for minimal overhead)
- **Posted Timer Interrupt** mechanism: `kvm_can_post_timer_interrupt()` conditions (`pi_inject_timer` + APICv + HLT-in-guest), complete delivery flow eliminating VM Exit entirely
- **Adaptive Timer Advance** algorithm: `adjust_lapic_timer_advance()` step-by-step tuning (initial 1us, max 5us, step 1/8), `kvm_wait_lapic_expire()` busy-wait for precision, `__wait_lapic_expire()` implementation
- **HLT/MWAIT-in-guest** synergy: `CPU_BASED_HLT_EXITING` bit clearing, interaction with posted timer interrupt
- **ASCII decision tree** showing timer backend selection (hrtimer → hv_timer → posted timer interrupt)
- **Module parameters** table (`lapic_timer_advance`, `preemption_timer`, `pi_inject_timer`)
- **Enhanced future outlook**: residual VM Exit analysis (TSC_DEADLINE MSR write, nested virt, periodic mode), Confidential Computing timer challenges (TDX/SEV-SNP trust model), optimal real-time configuration stack

## [2026-04-10] ingest | Two articles from raw/articles/

Ingested two presentation PDFs on KVM VM Exit optimization techniques.

**Source summaries (2 new):**
- [src-lcna-co2012-sekiyama](sources/src-lcna-co2012-sekiyama.md) — "Improvement of Real-time Performance of KVM" by Tomoki Sekiyama (Hitachi, LinuxCon 2012). CPU isolation via `KVM_SET_SLAVE_CPU`, direct interrupt delivery (disable external interrupt exiting), NMI-based host→guest IPI, direct EOI via x2APIC MSR passthrough, vector remapping for passed-through devices
- [src-minimizing-vmexits-pv-ipi-passthrough-timer](sources/src-minimizing-vmexits-pv-ipi-passthrough-timer.md) — "Minimizing VMExits by Aggressive PV IPI and Passthrough Timer" by Huaqiao & Yibo Zhou (ByteDance, ~2020). Timer passthrough (direct physical LAPIC timer access, host timer offloading to preemption timer), NoExit PVIPI (pi_desc passthrough, MSR.ICR non-intercept, guest-level posted-interrupt IPI)

**Updated entity pages (2):**
- [kvm-interrupt-virtualization](entities/kvm-interrupt-virtualization.md) — Added "Direct Interrupt Delivery" section (Sekiyama: CPU isolation + VMCS pin-based controls + NMI signaling + vector remapping + direct EOI), "NoExit PV IPI" section (ByteDance: pi_desc passthrough + MSR.ICR non-intercept + guest-side posted-interrupt IPI flow + performance data), updated interrupt delivery summary with 2 new paths
- [kvm-performance-tuning](entities/kvm-performance-tuning.md) — Added "CPU Isolation (Extreme RT Mode)" subsection under CPU Pinning

**Updated analysis pages (1):**
- [analysis-vm-exit-reduction-and-timer-virtualization](analyses/analysis-vm-exit-reduction-and-timer-virtualization.md) — Added section "三点五" on ByteDance Timer Passthrough: physical LAPIC timer direct access, TSC offset adjustment, host timer offloading, comparison table with 6 approaches, memcached +35.5% / cyclictest -30% benchmarks, NoExit PVIPI IPI cost comparison (11486→412 cycles). Updated future outlook to reference Timer Passthrough and Sekiyama's CPU isolation

## [2026-04-10] update | Deep restructuring of VM Exit analysis with ingested article content

Restructured and enriched the VM Exit reduction and timer virtualization analysis page based on the two newly ingested articles (Sekiyama 2012, ByteDance ~2020).

**Structural changes:**
- Renamed "三点五" section to proper "阶段九: Timer Passthrough" and "阶段十: NoExit PVIPI", relocated under section 3.2 (Timer VM Exit 优化的技术演进) alongside stages 一-八
- Updated five-generation timeline in section 二 to include "前沿 (~2020)" entry for Timer Passthrough and NoExit PVIPI
- Updated ASCII decision tree in section 3.3 to include Timer Passthrough as a fourth timer backend path

**Content additions:**
- Added "行业方案对比" subsection under 阶段九 with Tencent (Wanpeng Li exitless timer/IPI), Alibaba (Yang Zhang PV timer), and ByteDance (Timer Passthrough + NoExit PVIPI) competitive landscape
- Added Timer Passthrough and NoExit PVIPI rows to section 3.4 comparison table
- Added new section 4.5 (硬件直通方案的安全挑战): pi_desc exposure risks, EPTP Switch/VMFUNC hardening, physical timer contention, CoCo interaction
- Expanded section 4.6 (极致低延迟场景) with Sekiyama's specific RT use cases (factory automation, embedded systems, automated trading, HPC) and two-scenario configuration stack (通用低延迟 vs. 极致低延迟)
- Added IPI VM Exit bullet to section 4.1 (残余 VM Exit) referencing NoExit PVIPI

## [2026-04-14] ingest | Batch ingest of raw/articles (3 new sources + 1 corrupted)

Ingested three new sources from `raw/articles/` covering VM Exit reduction techniques from Chinese cloud vendors. One file (`tencent_solution_VMExit.pdf`) was found to be corrupted (invalid PDF header) — its content is partially covered by `all_solution_VMExit.pdf`.

**Source summaries created:**
- [src-all-solution-vmexit](sources/src-all-solution-vmexit.md) — Zhihu article by @惠伟: kvm_stat diagnosis of timer VM Exits (93% external interrupt from local timer, 62.5% MSR_WRITE from TSC_DEADLINE), comparison of three exit-less timer solutions from Tencent, Alibaba, and ByteDance
- [src-bytedance-solution-vmexit](sources/src-bytedance-solution-vmexit.md) — InfoQ article by Volcengine/ByteDance (2024-12): comprehensive edge high-performance VM architecture with four key techniques (interrupt non-exit, timer passthrough, VFIO interrupt bypass, IPI extreme fastpath) + kernel resource dynamic isolation. Benchmarks show >99% VM Exit reduction, 6-16% application throughput gains, CDN/streaming/acceleration production deployments
- [src-bytedance-solution-vmexit-code](sources/src-bytedance-solution-vmexit-code.md) — RFC kernel patch by dengqiao.joey/Yang Zhang (2020-09): full implementation of Passthrough IPI exposing pi_desc and physical ICR to guest, reducing IPI from 7K to 2K cycles. 14 files, +434 lines. Not merged upstream (security concerns)

**Concept page created:**
- [concept-exitless-timer](concepts/concept-exitless-timer.md) — Cross-cutting concept covering the three vendor approaches to eliminating timer VM Exits (Tencent posted-interrupt, Alibaba shared-page, ByteDance passthrough), prerequisites, and comparison

**Entity pages updated (frontmatter + source lists):**
- [kvm-interrupt-virtualization](entities/kvm-interrupt-virtualization.md) — Added new source references and tags
- [kvm-performance-tuning](entities/kvm-performance-tuning.md) — Added new source references and tags

**Analysis page updated:**
- [analysis-vm-exit-reduction-and-timer-virtualization](analyses/analysis-vm-exit-reduction-and-timer-virtualization.md) — Added new sources to frontmatter

**Note:** `tencent_solution_VMExit.pdf` has invalid PDF header — could not be ingested. Tencent's exit-less timer approach is documented via the all_solution survey article instead.

## [2026-04-14] update | analysis-vm-exit-reduction-and-timer-virtualization.md

Major update integrating content from three newly ingested sources. Page grew from ~566 to ~766 lines.

**Section 一 (VM Exit 代价)**:
- Added real-world `kvm_stat` diagnosis table from @惠伟: MSR_WRITE (95K, 62.5% from TSC_DEADLINE), EXTERNAL_INTERRUPT (35K, 93% from local timer), EPT_MISCONFIG (12K)
- Added diagnostic methodology (VM-Exit interruption info field analysis, MSR trace)

**Section 3.2 (行业方案对比)**:
- Expanded Alibaba PV Timer (Yang Zhang) description: shared-page mechanism, 7-part RFC patch series, trade-offs (eliminates MSR exit but wastes CPU on polling)
- Added Tencent exitless timer patch details ([v7,1/2] + [v7,2/2])
- Enhanced comparison table with "额外 CPU 开销" and "上游状态" columns

**Section 3.2 阶段十 (NoExit PVIPI)**:
- Added complete `pvipi_msr` union structure and MSR definitions from RFC patch
- Added guest-side fast path (`kvm_send_ipi()`) step-by-step: wrmsr_fence → set PIR → set ON → read nv/ndst → write physical ICR
- Added host-side implementation: pi_desc pointer change, pi_desc_setup(), PVIPI_PAGE_PRIVATE_MEMSLOT, MSR interception toggle
- Added mailing list discussion: Wanpeng Li security concerns, Yang Zhang TikTok production results
- Added dual benchmark comparison (KVM Forum 412 cycles vs RFC patch 2K cycles)
- Added limitation: max 128 vCPU

**New section 五 (Volcengine 边缘高性能虚拟机)**:
- §5.1 Architecture: design goals, control-plane/data-plane CPU split diagram
- §5.2 Four key technologies: (1) Guest interrupt non-exit (INTR_EXITING + host IRQ migration + IPI-as-NMI), (2) Timer passthrough cross-reference, (3) VFIO interrupt bypass (direct IOMMU IRTE modification bypassing Posted-Interrupt), (4) IPI Extreme Fastpath (assembly-optimized VMM emulation, distinct from NoExit PVIPI)
- §5.3 Dynamic kernel isolation framework: table comparing static (nohz_full/isolcpus cmdline) vs dynamic (enter/exit Guest) isolation for timer/interrupt/process dimensions
- §5.4 Micro benchmarks: IPI -22%, broadcast IPI -17%, timer -12.5%, TSC_DEADLINE write -89%, VM Exit count >99% reduction
- §5.5 Macro benchmarks: wrk +6%, Apache +12%, Redis +16%, netperf +8-13%
- §5.5 Production deployments: CDN -13.9-23.2% CPU, streaming -23.7% CPU, acceleration latency matching bare metal

**Section 4.6 (低延迟配置)**:
- Added 场景三 (Volcengine edge high-perf VM) configuration stack alongside existing 场景一/二

**Section 3.4 (技术现状总结表)**:
- Added rows for 阿里 PV Timer and Volcengine 边缘高性能 VM

**See also**: Added concept-exitless-timer and three new source page links

## [2026-04-14] create | Timer VM Exit 优化方案综述

Created a new standalone survey page synthesizing all Timer-caused VM Exit optimization approaches:

**New analysis page:**
- [analysis-timer-vmexit-optimization-survey](analyses/analysis-timer-vmexit-optimization-survey.md) — Comprehensive survey covering: problem definition with production kvm_stat data, 10-stage optimization evolution (tickless → TSC-deadline → preemption timer → posted timer interrupt → adaptive advance → MSR bitmap → kvmclock → HLT-in-guest → Timer Passthrough → NoExit PVIPI), side-by-side comparison of Tencent/Alibaba/ByteDance exit-less timer approaches, KVM timer data structures and code paths, Volcengine production deployment results, and future directions (hardware evolution, CoCo challenges, security attack surface)

**Updated:**
- [index.md](index.md) — Added new analysis page entry

## [2026-04-11] ingest | VMExit_opt_Hitachi_Sekiyama.md

Ingested a Chinese-language technical analysis of Sekiyama's 2012 Hitachi direct interrupt delivery scheme. This is a secondary analysis of the same work already covered by `src-lcna-co2012-sekiyama` (original LinuxCon slides), but adds an explicit six-step construction, a comparison table, and a detailed pros/cons evaluation highlighting disadvantages not captured in the original source.

**Source summary created:**
- [src-vmexit-opt-hitachi-sekiyama](sources/src-vmexit-opt-hitachi-sekiyama.md) — Chinese-language analysis: six-step direct interrupt delivery construction (CPU isolation → KVM_SET_SLAVE_CPU → disable external interrupt exiting → IRQ routing → NMI fallback → direct EOI), comparison table, pros/cons (debugging opacity, system hang risk from IRQ misconfiguration), framing as APICv software predecessor

**Entity pages updated:**
- [kvm-interrupt-virtualization](entities/kvm-interrupt-virtualization.md) — Added "Advantages and Limitations" subsection to Direct Interrupt Delivery section: non-mainline code, no overcommitment, debugging opacity, system hang risk from IRQ misrouting. Added historical positioning as APICv software predecessor. Added new source to frontmatter

**Updated:**
- [index.md](index.md) — Added new source summary entry

## [2026-04-11] create | 直接中断投递深度分析 (Sekiyama)

Created a dedicated analysis page providing a comprehensive deep-dive into Sekiyama's direct interrupt delivery approach.

**New analysis page:**
- [analysis-direct-interrupt-delivery-sekiyama](analyses/analysis-direct-interrupt-delivery-sekiyama.md) — Six-section analysis: (1) problem definition with 3-VM-Exit interrupt path ASCII diagram, (2) six-step construction with ASCII architecture diagrams (CPU isolation → KVM_SET_SLAVE_CPU → disable external interrupt exiting → IRQ routing → NMI fallback → direct EOI), (3) interrupt path comparison table (traditional vs Sekiyama vs APICv), (4) detailed pros/cons (debugging opacity, system hang risk, no overcommit, non-mainline), (5) target scenarios (factory automation, embedded, trading, HPC), (6) historical positioning as APICv software predecessor with design-pattern continuity table (CPU isolation → nohz_full, direct delivery → posted interrupts, NMI fallback → IPI-as-NMI)

**Updated:**
- [index.md](index.md) — Added new analysis page entry

## [2026-04-11] lint | Health check and fixes

Ran full wiki health check (89 pages). Found and fixed 4 issues:

**HIGH — Duplicate section removed:**
- [kvm-interrupt-virtualization](entities/kvm-interrupt-virtualization.md) — Removed duplicate "Interrupt Delivery Summary" section at line 211 (5 paths). The authoritative version at line 299 (7 paths, including Direct delivery and NoExit PVIPI) is retained. Page dropped from 318 → 308 lines.

**MEDIUM — Analysis pages renamed to follow `analysis-<name>.md` convention:**
- `analyses/vfio-device-passthrough.md` → `analyses/analysis-vfio-device-passthrough.md`
- `analyses/vfio-device-passthrough-zh.md` → `analyses/analysis-vfio-device-passthrough-zh.md`
- Updated all inbound links (10 files: index.md, concept-virtio-data-plane.md, concept-hardware-virtualization.md, src-bytedance-solution-vmexit.md, kvm-networking.md, analysis-vm-exit-reduction-and-timer-virtualization.md ×2, analysis-timer-vmexit-optimization-survey.md, analysis-vfio-device-passthrough.md, analysis-vfio-device-passthrough-zh.md)

**LOW — Overview count corrected:**
- [overview.md](overview.md) — "five specialized articles" → "six specialized articles", added mention of the Sekiyama analysis source

## [2026-04-11] lint | Health check and fixes (second pass)

Ran full wiki health check (101 pages). Found and fixed 5 issues:

**HIGH — Orphan analysis page integrated:**
- `analyses/epyc_based_kernel_lapic_time_analyze.md` → `analyses/analysis-epyc-lapic-timer.md` — Renamed to follow `analysis-<name>.md` convention, added YAML frontmatter (type, created, updated, sources, tags), added to index.md

**MEDIUM — 3 broken links in log.md fixed:**
- Line 228: `analyses/vfio-device-passthrough.md` → `analyses/analysis-vfio-device-passthrough.md`
- Line 263: `analyses/vfio-device-passthrough-zh.md` → `analyses/analysis-vfio-device-passthrough-zh.md`
- Line 267: `analyses/vfio-device-passthrough.md` → `analyses/analysis-vfio-device-passthrough.md`

**LOW — Standardized `title` field in 3 source files:**
- [src-all-solution-vmexit](sources/src-all-solution-vmexit.md) — Added `title` field
- [src-bytedance-solution-vmexit](sources/src-bytedance-solution-vmexit.md) — Added `title` field
- [src-bytedance-solution-vmexit-code](sources/src-bytedance-solution-vmexit-code.md) — Added `title` field

**Noted (not fixed):**
- 10 pages exceed 300-line guideline (largest: 766 lines). Analysis pages are long by nature; splitting deferred

## [2026-04-11] ingest | KVM-devirt: Zero-overhead Partition Hypervisor (KVM Forum 2022)

Ingested a 19-slide presentation by Liang Deng (ByteDance STE) from KVM Forum 2022. Introduces KVM-devirt, which extends KVM into a zero-overhead partition hypervisor by eliminating all VM Exits and address translations after guest init. Six techniques: interrupt/IPI/timer passthrough, memory/DMA devirtualization, virtio notification passthrough. Benchmarks show 20-30% improvement over standard VM, within 1% of native for single partition, and 9% faster than native with 4 partitions.

**Source summary created:**
- [src-kvm-devirt-kvmforum2022](sources/src-kvm-devirt-kvmforum2022.md) — Liang Deng, ByteDance STE (KVM Forum 2022): KVM-devirt zero-overhead partition hypervisor

**Entity page created:**
- [kvm-devirt](entities/kvm-devirt.md) — KVM-devirt architecture: six passthrough/devirtualization techniques, benchmarks, platform support

**Concept page created:**
- [concept-memory-devirtualization](concepts/concept-memory-devirtualization.md) — Memory de-virtualization: PV page table interfaces (set_pgd/pte_val), gfn-to-pfn/pfn-to-gfn mapping, EPT/NPT elimination, single-level translation

**Existing pages updated:**
- [concept-hardware-virtualization](concepts/concept-hardware-virtualization.md) — Added "Beyond Hardware Assistance: KVM-devirt" subsection, updated sources and See also
- [concept-exitless-timer](concepts/concept-exitless-timer.md) — Added KVM-devirt timer passthrough context paragraph, updated sources and See also
- [index.md](index.md) — Added 3 new page entries (source, entity, concept)
- [overview.md](overview.md) — Updated virtualization coverage with KVM-devirt partition hypervisor

## [2026-04-11] create | KVM-devirt 深度分析

Created a comprehensive analysis page synthesizing KVM-devirt with all existing wiki sources on VM Exit optimization.

**New analysis page:**
- [analysis-kvm-devirt-partition-hypervisor](analyses/analysis-kvm-devirt-partition-hypervisor.md) — Nine-section deep dive: (1) consolidation vs partitioning paradigm shift with Amdahl's Law analysis of the "faster than native" result, (2) six-technique overhead decomposition with quantified breakdown (memory devirt 14% > interrupt+IPI+timer 8%), (3) three-generation interrupt passthrough evolution (Sekiyama 2012 → Volcengine 2020 → KVM-devirt 2022), IPI passthrough dual-path comparison (NoExit PVIPI vs KVM-devirt), timer passthrough lineage, (4) memory devirtualization as key innovation: frequency×cost analysis explaining why EPT overhead > VM Exit overhead, "reverse paravirtualization" design pattern, shadow PT comparison, static memory tradeoffs, (5) KVM-devirt vs Volcengine comparison (zero overhead vs deployable transparency), (6) security model analysis (vector separation, PV page table isolation guarantees, DMA passthrough risks), (7) residual VM Exit enumeration, (8) future outlook (upstream challenges, CoCo conflicts, live migration obstacles, CXL interaction), (9) four key conclusions

**Updated:**
- [index.md](index.md) — Added analysis page entry

## [2026-04-11] create | LAPIC One-shot vs TSC-Deadline 精度差别研究

Created a deep-dive analysis comparing LAPIC timer one-shot mode and TSC-Deadline mode precision characteristics.

**New analysis page:**
- [analysis-lapic-oneshot-vs-tscdeadline-precision](analyses/analysis-lapic-oneshot-vs-tscdeadline-precision.md) — Nine-section analysis covering: (1) hardware working principles of both modes (bus clock decrement counter vs TSC absolute comparator), (2) quantified precision comparison (resolution 10-200x, jitter 2-5x, programming latency ~2x advantage for TSC-Deadline), (3) root cause analysis (clock domain difference: core TSC 2-5 GHz vs bus clock 100-400 MHz; absolute vs relative timing semantics; calibration error propagation), (4) Linux kernel implementation (lapic_next_event vs lapic_next_deadline, clockevents rating 100 vs 600, hrtimer integration), (5) KVM virtualization impact (VM-Exit overhead ~1μs masks ~10ns precision difference; timer passthrough restores hardware precision gap), (6) practical scenario analysis (HFT, real-time control, general server workloads), (7) why modern systems universally use TSC-Deadline

**Updated:**
- [index.md](index.md) — Added analysis page entry

## [2026-04-11] create | Linux 内核中断类型总览

Created a comprehensive Chinese-language analysis page classifying Linux kernel interrupt types across seven dimensions.

**New analysis page:**
- [analysis-interrupt-types-overview-zh](analyses/analysis-interrupt-types-overview-zh.md) — 多维度分类：同步/异步中断、异常三子类（Fault/Trap/Abort）、中断控制器（PIC/I/O APIC/LAPIC/MSI/IPI）、触发模式（边沿/电平）、IDT 门类型（中断门/陷阱门/系统门）、上下半部机制（Softirq/Tasklet/Work Queue）、KVM 虚拟化中断路径（PIC 模拟→APICv→Direct Delivery→NoExit PVIPI）

**Updated:**
- [index.md](index.md) — Added analysis page entry

## [2026-04-11] create | AVIC 是否需要 Guest Kernel 修改

Created a Chinese-language analysis page examining whether AMD AVIC requires guest kernel changes (conclusion: no).

**New analysis page:**
- [analysis-avic-guest-kernel-changes-zh](analyses/analysis-avic-guest-kernel-changes-zh.md) — 分析 AVIC/x2AVIC 对 guest 的完全透明性：AVIC 三大硬件组件（Backing Page、AVIC Table、Doorbell）的 guest 不可见性、timer MSR 强制拦截的 host 端处理、x2AVIC (Zen 4+) 同样无需 guest 修改、与五类需要 guest 修改方案的详细对比（KVM PV IPI、NoExit PVIPI、PV EOI/TLB/调度、KVM-devirt BM）、guest 配置调优建议

**Updated:**
- [index.md](index.md) — Added analysis page entry

## [2026-04-18] create | AMD PMC 溢出处理机制

Created a Chinese-language analysis page on AMD PMC overflow handling: hardware bit-47 rollover detection, host perf driver per-counter polling, KVM emulated PMU overflow path, and Zen 4+ GlobalStatus improvements.

**New analysis page:**
- [analysis-amd-pmc-overflow](analyses/analysis-amd-pmc-overflow.md) — AMD PMC 溢出处理全链路：硬件层 bit47 翻转检测与 NMI 投递、amd_pmu_handle_irq() 逐计数器轮询、KVM 模拟 PMU 溢出路径（kvm_perf_overflow → global_status → PMI 注入）、Zen 4+ PerfCntrGlobalStatus 系列 MSR 改进、AMD vs Intel 差异对比

**Updated:**
- [index.md](index.md) — Added analysis page entry

## [2026-04-18] update | AMD PMC 溢出处理——Zen 4+ Host/Guest 深度扩展

Major expansion of the AMD PMC overflow analysis with comprehensive Zen 4+ PerfMonV2 coverage for both host and guest VM paths.

**Updated analysis page:**
- [analysis-amd-pmc-overflow](analyses/analysis-amd-pmc-overflow.md) — Added: CPUID 8000_0022 PerfMonV2 detection, MSR addresses (0xC0000300-303), GlobalCtl AND PerfEvtSel.EN semantics, amd_pmu_v2_init() initialization, amd_pmu_v2_handle_irq() five-step NMI handler with freeze/thaw, Zen3- vs Zen4+ side-by-side comparison, Guest VM full timeline (t0-t3) with per-step VM-Exit accounting (6-16 exits per overflow), KVM global_status software abstraction architecture, PerfMonV2 CPUID exposure to guest, mediated PMU gap analysis (missing VMCB atomic load/save fields), software-only mediated PMU race window analysis, future VMCB extension roadmap, three-layer comparison table

**Updated:**
- [index.md](index.md) — Updated analysis page description

## [2026-04-18] create | RDPMC 不拦截对 Guest 溢出处理的影响

Created analysis examining how non-intercepted RDPMC affects guest PMC overflow handling across emulated and mediated PMU modes.

**New analysis page:**
- [analysis-rdpmc-passthrough-overflow-impact](analyses/analysis-rdpmc-passthrough-overflow-impact.md) — 核心不变量分析（硬件计数器所有权决定 RDPMC 正确性）、模拟模式下六步溢出处理损坏推演（delta 错误→bit47 误判→重装漂移→采样损坏）、pmc_read_counter() 三源合成机制（base+emulated+perf_event_read）、中介模式 RDPMC 正确性证明、AMD 未来 mediated PMU 场景推演、四象限决策矩阵

**Updated:**
- [index.md](index.md) — Added analysis page entry
