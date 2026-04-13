---
type: index
created: 2026-04-08
updated: 2026-04-09
---

# Wiki Index

Content catalog for the Kernel Wiki. Organized by page type.

## Overview

- [overview](overview.md) — High-level map of wiki coverage and key findings

## Source Summaries

- [src-understanding-the-linux-kernel](sources/src-understanding-the-linux-kernel.md) — Bovet & Cesati, 3rd Edition (Linux 2.6.11, x86)
- [src-linux-kernel-in-a-nutshell](sources/src-linux-kernel-in-a-nutshell.md) — Kroah-Hartman (Linux 2.6.x, practical build/configure/install guide)
- [src-advanced-linux-programming](sources/src-advanced-linux-programming.md) — Mitchell, Oldham, Samuel (GNU/Linux application development, user-space perspective)
- [src-understanding-linux-network-internals](sources/src-understanding-linux-network-internals.md) — Benvenuti (Linux 2.6, networking stack internals: devices, IPv4, ARP, routing)
- [src-professional-linux-kernel-architecture](sources/src-professional-linux-kernel-architecture.md) — Mauerer (Linux 2.6.24, comprehensive internals: CFS, namespaces, SLUB, TCP/UDP, netfilter)
- [src-qemu-kvm-source-code-and-application](sources/src-qemu-kvm-source-code-and-application.md) — Li Qiang (QEMU 2.8.1 / Linux 4.4.161, QEMU/KVM source-code-level analysis: CPU/memory/interrupt/IO virtualization)
- [src-mastering-kvm-virtualization](sources/src-mastering-kvm-virtualization.md) — Chirammal, Mukhedkar, Vettathu (KVM operations: libvirt management, networking, storage, migration, performance tuning)
- [src-debugging-with-gdb](sources/src-debugging-with-gdb.md) — Stallman, Pesch, Shebs (GDB 7.6.50, Tenth Edition: complete debugger reference — breakpoints, watchpoints, tracepoints, reverse debugging, remote debugging, Python API, GDB/MI)

## Entities

### Process Management
- [process-management](entities/process-management.md) — Process descriptors, creation (fork/clone), context switching, destruction
- [process-scheduler](entities/process-scheduler.md) — Process scheduling: O(1) scheduler (2.6.0-2.6.22) and CFS (2.6.23+)
- [cfs-scheduler](entities/cfs-scheduler.md) — Completely Fair Scheduler: vruntime, red-black tree, sched_entity, group scheduling
- [system-calls](entities/system-calls.md) — System call mechanism: int $0x80, sysenter, sys_call_table, parameter verification
- [signals](entities/signals.md) — Signal generation, delivery, handler stack frames, real-time signals, job control
- [program-execution](entities/program-execution.md) — execve, ELF loading, dynamic linking, script handling

### Memory
- [memory-management](entities/memory-management.md) — Physical memory: buddy system, zones, slab allocator, kmalloc, vmalloc
- [process-address-space](entities/process-address-space.md) — Virtual memory: mm_struct, VMAs, page fault handling, COW, heap
- [page-cache](entities/page-cache.md) — Page cache: address_space, radix tree, buffer cache, read-ahead, writeback
- [page-frame-reclaiming](entities/page-frame-reclaiming.md) — PFRA: LRU lists, kswapd, reverse mapping, swap, OOM killer

### Filesystems and I/O
- [virtual-filesystem](entities/virtual-filesystem.md) — VFS layer: superblock, inode, dentry, file objects, dcache, pathname lookup
- [ext2-ext3](entities/ext2-ext3.md) — Ext2/Ext3 filesystems: block groups, indirect blocks, JBD journaling
- [extended-attributes-acls](entities/extended-attributes-acls.md) — Extended attributes (xattr namespaces) and POSIX Access Control Lists
- [block-layer](entities/block-layer.md) — Generic block layer: bio, request queues, I/O schedulers (Noop/Deadline/Anticipatory/CFQ)
- [device-driver-model](entities/device-driver-model.md) — Device model: kobject, sysfs, bus/device/driver, PCI, DMA

### Concurrency
- [pthreads](entities/pthreads.md) — POSIX threads: creation, synchronization (mutexes, semaphores, condition variables), cancellation, TSD

### Virtualization and Isolation
- [namespaces](entities/namespaces.md) — Linux namespaces: nsproxy, PID/UTS/IPC/mount/network/user namespaces, CLONE_NEW* flags, container building blocks
- [qemu-kvm-overview](entities/qemu-kvm-overview.md) — QEMU/KVM architecture: QOM type system, QDev device model, event loop, HMP/QMP monitor, thread model
- [kvm-cpu-virtualization](entities/kvm-cpu-virtualization.md) — CPU virtualization: VMX modes, VMCS, VCPU lifecycle, kvm_cpu_exec loop, kvm_run shared data
- [kvm-memory-virtualization](entities/kvm-memory-virtualization.md) — Memory virtualization: EPT, shadow page tables, KVM memory slots, MMIO, dirty page tracking
- [kvm-interrupt-virtualization](entities/kvm-interrupt-virtualization.md) — Interrupt virtualization: PIC/IOAPIC/LAPIC emulation, MSI, APICv/posted interrupts, IRQfd/IOeventfd
- [virtio-framework](entities/virtio-framework.md) — Virtio I/O framework: VRing, VirtQueue, VirtIODevice, PCI transport, virtio-net, virtio-blk
- [vhost](entities/vhost.md) — Vhost kernel-based virtio backend: data plane in kernel, vhost-net, ioeventfd/irqfd, vhost_worker thread
- [vm-live-migration](entities/vm-live-migration.md) — VM live migration: migration phases, dirty page sync, XBZRLE, post-copy, VMState serialization
- [qemu-machine-emulation](entities/qemu-machine-emulation.md) — QEMU machine emulation: i440FX/PIIX3 chipset, SeaBIOS, fw_cfg, ACPI, VGA, timers, serial
- [libvirt-management](entities/libvirt-management.md) — libvirt management: libvirtd, virsh CLI, virt-install, virt-manager, domain XML, QMP, monitoring/logging
- [kvm-networking](entities/kvm-networking.md) — KVM networking: Linux bridge, libvirt networks, Open vSwitch, macvtap, SR-IOV, nwfilter
- [kvm-storage](entities/kvm-storage.md) — KVM storage: pools/volumes, raw/qcow2 formats, qemu-img, iSCSI/Ceph/GlusterFS, cache modes, virtio-blk vs virtio-scsi
- [kvm-performance-tuning](entities/kvm-performance-tuning.md) — KVM performance tuning: CPU pinning, NUMA, hugepages, KSM, I/O tuning, vhost-net, tuned profiles
- [vm-snapshots-templates](entities/vm-snapshots-templates.md) — VM snapshots and templates: virt-sysprep, virt-clone, internal/external snapshots, blockcommit/blockpull
- [v2v-p2v-migration](entities/v2v-p2v-migration.md) — V2V/P2V migration: virt-v2v (VMware/Xen/Hyper-V to KVM), virt-p2v (physical-to-virtual)
- [vfio-device-passthrough](entities/vfio-device-passthrough.md) — VFIO framework, IOMMU groups, DMA/interrupt remapping, PCI/SR-IOV device passthrough to KVM guests

### Infrastructure
- [interrupt-handling](entities/interrupt-handling.md) — IDT, PIC/APIC, do_IRQ, softirqs, tasklets, work queues
- [timing-subsystem](entities/timing-subsystem.md) — Hardware timers, jiffies, timer wheel, delay calibration, NTP
- [ipc](entities/ipc.md) — Pipes, FIFOs, System V IPC, POSIX message queues, UNIX domain sockets
- [proc-filesystem](entities/proc-filesystem.md) — /proc virtual filesystem: process entries, hardware/kernel/system info, sysctl
- [boot-process](entities/boot-process.md) — BIOS -> bootloader -> setup -> startup_32 -> start_kernel -> init, boot parameters
- [kernel-modules](entities/kernel-modules.md) — Loadable modules: insmod/rmmod, symbol export, demand loading, installation
- [auditing](entities/auditing.md) — Audit subsystem: audit_context, netlink communication, rules, SELinux AVC integration

### Networking
- [sk-buff](entities/sk-buff.md) — Socket buffer (sk_buff): the fundamental packet representation, buffer management, cloning
- [net-device](entities/net-device.md) — Network device (net_device): interface representation, registration, state, operations
- [network-device-initialization](entities/network-device-initialization.md) — Notification chains, PCI NIC probing, net_dev_init(), device registration lifecycle
- [frame-reception](entities/frame-reception.md) — NAPI and legacy frame reception, softnet_data, net_rx_action, protocol demultiplexing
- [frame-transmission](entities/frame-transmission.md) — dev_queue_xmit, hard_start_xmit, TX flow control, NET_TX_SOFTIRQ completion
- [bridging](entities/bridging.md) — L2 bridging: STP (802.1D), forwarding database, MAC learning, bridge device, brctl
- [ipv4-subsystem](entities/ipv4-subsystem.md) — IPv4: ip_rcv/ip_forward/ip_local_deliver, transmission, fragmentation/defrag, ICMP, L4 dispatch
- [neighboring-subsystem](entities/neighboring-subsystem.md) — Protocol-independent L3-to-L2 address resolution, NUD state machine, neigh_table/neighbour structures
- [arp](entities/arp.md) — Address Resolution Protocol: packet format, gratuitous ARP, proxy ARP, arp_filter, ARPD
- [routing-subsystem](entities/routing-subsystem.md) — IPv4 routing architecture: two-level cache+FIB design, route types, scopes, input/output routing
- [routing-cache](entities/routing-cache.md) — Routing cache: hash table, lookup/insertion, garbage collection, ICMP redirect handling
- [routing-tables-fib](entities/routing-tables-fib.md) — FIB: fib_table/fib_info/fib_nh/fib_node/fib_alias structures, fn_hash and fn_trie backends, route addition/deletion
- [socket-layer](entities/socket-layer.md) — Socket layer: BSD socket API, struct socket/sock/proto_ops/proto, socket creation flow, send/recv data paths, buffer flow control
- [tcp-udp](entities/tcp-udp.md) — TCP/UDP transport: tcp_sock, congestion control, three-way handshake, udp_sock, sendmsg/rcv paths
- [netfilter](entities/netfilter.md) — Netfilter: 5 hook points, nf_hook_ops, iptables, connection tracking, NAT

### Devices and Hardware
- [devices](entities/devices.md) — Device types, device nodes, /dev entries, ioctl, PTYs, special devices
- [usb-subsystem](entities/usb-subsystem.md) — USB subsystem: layered architecture (HCD/core/drivers), URBs, device enumeration, host controllers

### Build and Configuration
- [kernel-build-system](entities/kernel-build-system.md) — Kbuild: make targets, parallel/cross-compilation, ccache, distcc
- [kernel-configuration](entities/kernel-configuration.md) — Kconfig: menuconfig, xconfig, oldconfig, .config, device discovery workflow
- [gnu-toolchain](entities/gnu-toolchain.md) — GCC, GNU Make, GDB, strace, valgrind, profiling tools
- [gdb-debugger](entities/gdb-debugger.md) — GDB debugger: breakpoints, watchpoints, catchpoints, stepping, data examination, reverse execution, tracepoints, Python API, TUI
- [gdb-remote-debugging](entities/gdb-remote-debugging.md) — GDB remote debugging: gdbserver, Remote Serial Protocol (RSP), agent expressions, target descriptions, .gdb_index

## Concepts

- [concept-kernel-synchronization](concepts/concept-kernel-synchronization.md) — Per-CPU vars, atomics, barriers, spinlocks, RCU, semaphores, BKL
- [concept-copy-on-write](concepts/concept-copy-on-write.md) — COW in fork, do_wp_page, anonymous zero-page, demand paging
- [concept-memory-addressing](concepts/concept-memory-addressing.md) — Segmentation, paging modes, four-level page tables, TLB, kernel memory layout
- [concept-linux-security](concepts/concept-linux-security.md) — Users/groups, permissions, capabilities, set-UID, chroot, PAM
- [concept-inline-assembly](concepts/concept-inline-assembly.md) — GCC asm construct, GAS syntax, constraints, volatile, rdtsc/cpuid examples
- [concept-napi-interrupt-mitigation](concepts/concept-napi-interrupt-mitigation.md) — NAPI: interrupt+polling hybrid for high-throughput reception, budget mechanism, livelock prevention
- [concept-policy-routing](concepts/concept-policy-routing.md) — Multiple routing tables, fib_rules, source/TOS/fwmark-based routing decisions
- [concept-network-packet-flow](concepts/concept-network-packet-flow.md) — End-to-end packet flow through the Linux networking stack (ingress and egress paths)
- [concept-scheduling-classes](concepts/concept-scheduling-classes.md) — Pluggable scheduling class framework: sched_class vtable, class hierarchy, dispatch mechanism
- [concept-page-mobility](concepts/concept-page-mobility.md) — Page mobility and anti-fragmentation: migration types, buddy system integration, fallback mechanism, ZONE_MOVABLE
- [concept-hardware-virtualization](concepts/concept-hardware-virtualization.md) — Hardware virtualization: VT-x/VMX, Type 1 vs Type 2 hypervisors, EPT, APICv, progressive VM Exit elimination
- [concept-virtio-data-plane](concepts/concept-virtio-data-plane.md) — Virtio data plane optimization: control vs data plane, IOThread, vhost kernel offloading, device passthrough
- [concept-reverse-debugging](concepts/concept-reverse-debugging.md) — Reverse debugging: process record/replay, reverse execution commands, checkpoints, hardware-assisted tracing

## Comparisons

- [cmp-slab-slub-slob](comparisons/cmp-slab-slub-slob.md) -- SLAB vs SLUB vs SLOB: trade-offs between the three Linux kernel slab allocator implementations

## Analyses

_No analyses yet._
