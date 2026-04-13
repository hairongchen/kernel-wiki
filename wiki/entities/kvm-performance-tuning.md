---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [mastering-kvm-virtualization]
tags: [kvm, performance, numa, hugepages, ksm, cpu-pinning, tuning]
---

# KVM Performance Tuning

Performance tuning for KVM spans CPU scheduling, memory allocation, I/O handling, and network configuration. Proper tuning can bring guest performance close to bare-metal levels. This page synthesizes techniques from *Mastering KVM Virtualization* (Chirammal et al., Packt, 2016).

## CPU Pinning

CPU pinning binds virtual CPUs to specific physical CPUs, reducing cache misses and cross-NUMA memory access. KVM/[libvirt-management](libvirt-management.md) supports three levels of pinning via `<cputune>`:

**vCPU pinning** ties individual guest vCPUs to host CPUs. Use `virsh vcpupin <domain> <vcpu> <cpuset>` at runtime, or in XML:

```xml
<cputune>
  <vcpupin vcpu="0" cpuset="2"/>
  <vcpupin vcpu="1" cpuset="3"/>
  <emulatorpin cpuset="0"/>
  <iothreadpin iothread="1" cpuset="4"/>
</cputune>
```

**Emulator pinning** (`emulatorpin`) pins the QEMU emulator process to a dedicated core, preventing it from competing with guest vCPUs. **IOThread pinning** (`iothreadpin`) pins I/O handler threads to specific cores, isolating disk I/O processing.

**Best practices**: pin all vCPUs, emulator, and IOThreads to cores on the **same NUMA node**. Use dedicated physical CPUs per guest; avoid overcommitting pinned cores. Verify with `virsh vcpuinfo <domain>`.

## NUMA Tuning

In a NUMA system each node consists of a socket (one or more CPU cores) plus locally attached memory. Accessing remote-node memory incurs significantly higher latency.

### Diagnostic Tools

| Tool | Purpose |
|------|---------|
| `numactl --hardware` | Node count, CPUs per node, memory per node |
| `numastat` / `numastat -c qemu-kvm` | Per-node memory stats (hits, misses, foreign) |
| `lstopo` | Graphical/text topology map (hwloc package) |

### numad and libvirt numatune

The `numad` daemon monitors NUMA topology and automatically places VMs on optimal nodes. It integrates with libvirt via `<numatune>` placement="auto". The `<numatune>` XML element sets the memory policy:

```xml
<numatune>
  <memory mode="strict" nodeset="0"/>
</numatune>
```

Modes: **strict** (fail if node unavailable), **preferred** (try specified node, fall back), **interleave** (round-robin across nodes).

### Guest NUMA Topology

For large guests spanning multiple host NUMA nodes, expose a guest-visible NUMA topology:

```xml
<cpu>
  <numa>
    <cell id="0" cpus="0-3" memory="4194304" unit="KiB"/>
    <cell id="1" cpus="4-7" memory="4194304" unit="KiB"/>
  </numa>
</cpu>
```

Pair with `<numatune>` to map each guest cell to a host node.

## Hugepages

Standard 4 KB pages cause significant TLB pressure. Hugepages reduce TLB misses using larger page sizes. See also [kvm-memory-virtualization](kvm-memory-virtualization.md) and [memory-management](memory-management.md).

**2 MB pages** can be allocated at runtime; **1 GB pages** typically require boot-time allocation.

```bash
# Runtime (2 MB)
echo 1024 > /proc/sys/vm/nr_hugepages
# Per-NUMA-node
echo 512 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
# Boot-time (1 GB) -- add to kernel cmdline:
#   hugepagesz=1G hugepages=4
```

Mount hugetlbfs (persistent via `/etc/fstab`):

```bash
mount -t hugetlbfs hugetlbfs /dev/hugepages
```

Enable for a guest via domain XML:

```xml
<memoryBacking>
  <hugepages>
    <page size="2048" unit="KiB"/>
  </hugepages>
</memoryBacking>
```

Per-NUMA-node hugepage sizes are supported by adding `nodeset` attributes to individual `<page>` elements (e.g., 1 GB pages on node 0, 2 MB on node 1).

**Transparent Huge Pages (THP)** automatically promote pages without explicit allocation (`/sys/kernel/mm/transparent_hugepage/enabled`). For KVM hosts, explicit hugepages are preferred over THP because THP can cause latency spikes during compaction.

## KSM (Kernel Same-page Merging)

KSM scans memory for identical pages across VMs and merges them copy-on-write, enabling memory overcommitment at the cost of CPU overhead.

### Tunables (`/sys/kernel/mm/ksm/`)

| File | Description | Default |
|------|-------------|---------|
| `run` | 0 = stopped, 1 = running | 0 |
| `pages_to_scan` | Pages examined per cycle | 100 |
| `sleep_millisecs` | Delay between cycles (ms) | 200 |

Higher `pages_to_scan` and lower `sleep_millisecs` merge faster but cost more CPU.

### Statistics and Savings

`pages_shared` = unique pages with duplicates; `pages_sharing` = total pages referencing shared copies. **Memory saved** = `(pages_sharing - pages_shared) * page_size`.

### Services

- `ksm` -- systemd service enabling/disabling the kernel KSM thread.
- `ksmtuned` -- daemon that dynamically adjusts tunables based on load and free memory (config: `/etc/ksmtuned.conf`).

### Trade-offs

- **CPU vs. memory**: scanning cost is proportional to total guest memory. Savings are largest when many guests run the same OS.
- **Security**: KSM is vulnerable to timing side-channel attacks (attacker detects page presence via write-fault latency). Disable on security-sensitive hosts.

## I/O Tuning

Disk I/O is often the primary virtualization bottleneck. See also [kvm-storage](kvm-storage.md).

### IOThreads

QEMU defaults to a single I/O thread. Dedicate threads per disk for parallelism:

```xml
<iothreads>2</iothreads>
<disk type="file" device="disk">
  <driver name="qemu" type="qcow2" iothread="1"/>
  ...
</disk>
```

Combine with `<iothreadpin>` for core-level placement (see CPU Pinning).

### I/O Scheduler Selection

- **Host**: use `deadline` or `noop` for VM-backing block devices (`cfq` adds unhelpful overhead).
- **Guest**: use `noop` since the host already schedules I/O.

Set via: `echo deadline > /sys/block/sda/queue/scheduler`.

### Block I/O Throttling and Weight

Limit per-device throughput/IOPS to prevent noisy-neighbor problems:

```bash
virsh blkdeviotune <domain> <device> --total-bytes-sec 104857600 --total-iops-sec 1000
```

Supports `--read-bytes-sec`, `--write-bytes-sec`, `--read-iops-sec`, `--write-iops-sec` granularity.

Set relative I/O priority with cgroup-based weight (100--1000): `virsh blkiotune <domain> --weight 500`.

## Network Tuning

See also [kvm-networking](kvm-networking.md).

**vhost-net**: the `vhost_net` kernel module moves virtio-net data-plane processing into the kernel, reducing latency and CPU overhead. Enabled by default for virtio-net interfaces; verify with `lsmod | grep vhost_net`.

**Multiqueue virtio-net**: spread network processing across CPUs for high-throughput VMs:

```xml
<interface type="network">
  <model type="virtio"/>
  <driver name="vhost" queues="4"/>
</interface>
```

Inside the guest: `ethtool -L eth0 combined 4`. Set queue count equal to guest vCPU count.

**SR-IOV**: passes a physical NIC's virtual function (VF) directly to the guest via PCI passthrough, delivering near-native performance. Trade-off: live migration is not supported with SR-IOV passthrough.

## Memory Ballooning

The virtio-balloon device dynamically adjusts guest memory without restart. The balloon driver inflates (reclaims guest pages for the host) or deflates (returns pages to the guest).

```bash
virsh setmem <domain> 2097152 --live     # set to 2 GB
virsh dommemstat <domain>                 # query stats: actual, unused, available, rss
```

Use `dommemstat` metrics for capacity planning and automated adjustment.

## CPU Model Selection

The CPU model exposed to guests affects performance, feature availability, and migration compatibility.

| Mode | Description | Migration |
|------|-------------|-----------|
| **host-passthrough** | Exact host CPU exposed | Only to identical hosts |
| **host-model** | Closest QEMU model selected | Same or superset features |
| **custom** | Manual model + features | Full control |

Fine-tune features with policy attributes (`require`, `disable`, `optional`, `force`, `forbid`):

```xml
<cpu mode="custom" match="exact">
  <model>Haswell</model>
  <feature policy="require" name="vmx"/>
  <feature policy="disable" name="svm"/>
</cpu>
```

Use `host-passthrough` for maximum single-host performance; `host-model` or `custom` for migration across heterogeneous hardware.

## tuned Profiles

The `tuned` daemon applies system-level optimization profiles.

- **virtual-host** -- hypervisor profile: enables KSM, configures hugepages, adjusts dirty ratios, sets deadline scheduler.
- **virtual-guest** -- guest profile: reduces timers, enables paravirtual clock.

```bash
tuned-adm list                  # available profiles
tuned-adm active                # current profile
tuned-adm profile virtual-host  # apply hypervisor profile
tuned-adm recommend             # suggested profile
```

## See also

- [libvirt-management](libvirt-management.md)
- [kvm-storage](kvm-storage.md)
- [kvm-networking](kvm-networking.md)
- [kvm-memory-virtualization](kvm-memory-virtualization.md)
- [memory-management](memory-management.md)
