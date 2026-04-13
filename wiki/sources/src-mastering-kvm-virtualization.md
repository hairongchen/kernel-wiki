---
type: source
created: 2026-04-09
updated: 2026-04-09
sources: [mastering-kvm-virtualization]
tags: [kvm, qemu, libvirt, virtualization, operations, source-summary]
---

# Mastering KVM Virtualization

**Authors**: Humble Devassy Chirammal, Prasad Mukhedkar, Anil Vettathu
**Publisher**: Packt Publishing, 2016
**ISBN**: 9781784399054
**Language**: English

A practical, operations-focused guide to KVM virtualization covering the full stack from hardware-assisted virtualization concepts through libvirt management, networking, storage, live migration, performance tuning, V2V/P2V conversion, and cloud integration with OpenStack. Unlike the source-code-level analysis in "QEMU/KVM Source Code Analysis and Application," this book focuses on the **management and operational perspective** — how to deploy, configure, monitor, and optimize KVM environments in production.

## Chapter-by-Chapter Summary

### Chapter 1: Understanding Linux Virtualization
Covers virtualization fundamentals (full, para, hardware-assisted, OS-level), hypervisor taxonomy (Type 1 vs Type 2), and the x86 virtualization challenge (17 sensitive-but-unprivileged instructions violating Popek-Goldberg requirements). Solutions: binary translation (VMware), paravirtualization (Xen), hardware-assisted (Intel VT-x / AMD-V). KVM's unique position as a kernel module that turns Linux into a Type 1 hypervisor. Overview of the ecosystem: QEMU, libvirt, virsh, virt-manager, oVirt, OpenStack.

### Chapter 2: KVM Internals
KVM kernel modules (`kvm.ko`, `kvm-intel.ko`/`kvm-amd.ko`). The `/dev/kvm` ioctl interface at three levels: system (KVM_CREATE_VM), VM (KVM_CREATE_VCPU, KVM_SET_USER_MEMORY_REGION), and vCPU (KVM_RUN, KVM_GET/SET_REGS). QEMU-KVM execution flow. VMExit reasons (KVM_EXIT_IO, KVM_EXIT_MMIO, KVM_EXIT_HLT). Memory virtualization: shadow page tables vs EPT/NPT. KSM (Kernel Same-page Merging). Memory ballooning via virtio-balloon. Hugepages (2MB/1GB). Virtio architecture (frontend/backend/virtqueue/vring). Vhost and vhost-user. NUMA awareness.

### Chapter 3: Setting Up Standalone KVM Virtualization
Package installation (RHEL/CentOS/Ubuntu). Verification (`lsmod`, `virt-host-validate`, `virsh nodeinfo`). VM creation with `virt-install` (all key parameters). VM configuration in `/etc/libvirt/qemu/`. `virsh` operations (list, start, shutdown, destroy, suspend, resume, console, dominfo). QEMU Monitor and QMP access via `virsh qemu-monitor-command`.

### Chapter 4: Networking in KVM
**Linux bridge**: `brctl`/`iproute2` commands, tap devices, how bridging works at L2. **libvirt networks**: NAT (default `virbr0`), routed, isolated, bridged modes with XML definitions. DHCP via dnsmasq. **Open vSwitch**: architecture (ovs-vswitchd, ovsdb-server, kernel module), VLANs (access/trunk ports), GRE/VXLAN tunnels, OpenFlow rules, port mirroring, libvirt integration. **macvtap**: VEPA/bridge/private/passthrough modes. **SR-IOV**: PF/VF concepts, IOMMU requirement, VF creation, PCI passthrough via `<hostdev>` or `<interface type='hostdev'>`. **Network filtering** (nwfilter): ebtables/iptables-based VM firewall. Multi-queue virtio-net, vhost-net, jumbo frames.

### Chapter 5: Storage in KVM
**libvirt storage architecture**: pools and volumes. **Disk image formats**: raw (no metadata, best perf), qcow2 (COW, backing files, snapshots, compression, encryption, preallocation modes, lazy_refcounts, cluster size). **qemu-img tool**: create, info, convert, resize, snapshot, check, rebase, commit, amend, compare, map. **Storage pool types**: directory, LVM, NFS, iSCSI (with CHAP auth), Ceph RBD (CephX, libvirt secrets), GlusterFS (native QEMU driver). **Disk configuration**: virtio-blk vs virtio-scsi comparison, cache modes (none/writethrough/writeback/directsync/unsafe), I/O modes (threads/native), disk hot-add/remove.

### Chapter 6: VM Lifecycle Management
VM states (defined, running, paused, saved, managed-save, crashed, PM-suspended). Lifecycle operations: define/create, start, shutdown/destroy, suspend/resume, save/restore, managed-save. Live configuration changes (hot-plug vCPUs, memory via balloon). **Templates**: `virt-sysprep` for generalization, `virt-builder` for quick image creation. **Cloning**: `virt-clone` (full clone), backing-file linked clones. **Snapshots**: internal (within qcow2, disk-only or with memory) vs external (overlay chains). Blockcommit (merge overlay into backing) and blockpull (flatten chain into active). Management best practices.

### Chapter 7: VM Migration
Migration types: offline, live (pre-copy), post-copy. Prerequisites (shared storage, CPU compatibility, network, matching versions). `virsh migrate` flags (`--live`, `--persistent`, `--undefinesource`, `--tunnelled`, `--p2p`, `--copy-storage-all/inc`, `--postcopy`, `--auto-converge`, `--compressed`). Migration modes: direct, P2P, tunnelled. Pre-copy algorithm (iterative dirty page transfer, convergence). Post-copy algorithm (userfaultfd-based, guaranteed convergence, split-state risk). Bandwidth/downtime tuning (`migrate-setmaxdowntime`, `migrate-setspeed`, `domjobinfo`). TLS certificates for secure migration. CPU compatibility checking (`cpu-compare`, `cpu-baseline`, CPU modes).

### Chapter 8: Performance Tuning and Optimization
**CPU**: vCPU pinning (`vcpupin`, `emulatorpin`, `<cputune>` XML), NUMA topology awareness (`numactl`, `numastat`, `numad`, `<numatune>` XML with strict/preferred/interleave modes, guest NUMA topology), CPU model selection (host-passthrough/host-model/custom). **Memory**: KSM (`/sys/kernel/mm/ksm/` tunables, `ksmtuned`), hugepages (2MB/1GB, hugetlbfs, THP), memory ballooning. **I/O**: IOThreads (`<iothreads>`), cache mode selection, I/O scheduler (deadline/noop for hosts, noop for guests), block I/O throttling (`blkdeviotune`), I/O weight (`blkiotune`). **Network**: vhost-net, multiqueue virtio-net, SR-IOV. **tuned** daemon profiles (`virtual-host`, `virtual-guest`).

### Chapters 9-13: Monitoring, Logging, Troubleshooting
**Monitoring**: `virt-top` (per-VM resource usage), `perf kvm stat` (VMExit analysis), SNMP/Nagios/Grafana integration with `collectd-virt` plugin. **Logging**: QEMU logs (`/var/log/libvirt/qemu/`), libvirt daemon logging (`libvirtd.conf` log_level/log_filters/log_outputs), runtime log changes via `virt-admin`. **Troubleshooting**: QEMU monitor commands via `virsh qemu-monitor-command --hmp`, common failure scenarios.

### Chapter 14: V2V and P2V Migration
**virt-v2v**: converts VMs from VMware/Xen/Hyper-V to KVM. Input modes (vmx, vpx, ova, xen+ssh, disk). Output modes (local, libvirt, rhev, glance). Conversion steps: disk format conversion, driver replacement (VMware Tools → virtio), boot config adjustment. **virt-p2v**: physical-to-virtual conversion via bootable ISO + conversion server architecture.

### Chapter 15: OpenStack Integration
Nova compute with libvirt driver (`LibvirtDriver`). Nova config (`nova.conf` [libvirt] section). Glance image formats. Neutron networking (ML2 plugin, OVS/Linux bridge agents, VXLAN/VLAN). Cinder block storage backends (LVM, Ceph RBD, NFS).

### Chapter 16: Automation
Cloud-init with NoCloud data source (seed ISO, user-data/meta-data). Terraform with `terraform-provider-libvirt` (`libvirt_domain`, `libvirt_volume`, `libvirt_network`). Ansible with `community.libvirt` collection (`virt`, `virt_net`, `virt_pool` modules). Production best practices (overcommit ratios, monitoring, backup, SELinux/sVirt).

## Key Themes

1. **Operations-first perspective**: Unlike source-code analyses, this book focuses on how to deploy, configure, and manage KVM from the sysadmin/DevOps perspective — virsh commands, libvirt XML, storage backends, and monitoring tools.
2. **libvirt as the management abstraction**: Nearly all operations go through libvirt rather than direct QEMU/KVM interaction, providing a consistent API across hypervisor types.
3. **Storage flexibility**: Comprehensive coverage of storage backends from simple directories through enterprise distributed storage (Ceph, GlusterFS), with detailed treatment of cache modes, I/O modes, and image formats.
4. **Network topology options**: From simple NAT to production OVS with VXLAN tunnels and SR-IOV passthrough, covering the full spectrum of virtual networking complexity.
5. **Performance as a configuration problem**: Most performance optimizations are achieved through configuration (CPU pinning, NUMA affinity, hugepages, cache modes, I/O schedulers) rather than code changes.

## Relationship to Other Sources

This book complements the existing wiki sources in two ways. It provides the **operational counterpart** to the source-code analysis in "QEMU/KVM Source Code Analysis and Application" (Li Qiang) — where Li Qiang shows how EPT page faults are handled in `handle_ept_violation()`, this book shows how to enable hugepages via `<memoryBacking>` XML. It also connects KVM to the broader Linux ecosystem covered by the other books: KSM uses the page merging infrastructure from the memory management subsystem; vhost-net builds on the networking stack; SR-IOV uses the PCI and IOMMU subsystems; I/O schedulers are the same ones documented in "Understanding the Linux Kernel."

## See also

- [libvirt-management](../entities/libvirt-management.md)
- [kvm-networking](../entities/kvm-networking.md)
- [kvm-storage](../entities/kvm-storage.md)
- [kvm-performance-tuning](../entities/kvm-performance-tuning.md)
- [vm-snapshots-templates](../entities/vm-snapshots-templates.md)
- [v2v-p2v-migration](../entities/v2v-p2v-migration.md)
- [vm-live-migration](../entities/vm-live-migration.md)
- [qemu-kvm-overview](../entities/qemu-kvm-overview.md)
