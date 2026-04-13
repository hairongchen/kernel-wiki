---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [mastering-kvm-virtualization]
tags: [kvm, storage, qcow2, libvirt, storage-pool, virtio-scsi]
---

# KVM Virtual Storage

KVM virtual storage encompasses the disk image formats, storage backends, and I/O configuration options that present block devices to guest virtual machines. The [libvirt-management](libvirt-management.md) framework provides a unified abstraction over diverse storage technologies through its pool/volume model.

## libvirt Storage Architecture

libvirt organizes storage into two abstractions:

- **Storage pools** -- containers that provide storage capacity. A pool maps to a directory, logical volume group, NFS export, iSCSI target, Ceph cluster, or other backend.
- **Storage volumes** -- individual units of storage allocated from a pool. A volume typically corresponds to a disk image file, logical volume, or RBD image.

This two-level model decouples guest disk definitions from the underlying storage technology, allowing migration between backends without reconfiguring VMs.

## Disk Image Formats

### raw

A byte-for-byte image of the virtual disk with no metadata layer. Offers the best I/O performance because the host can map guest offsets directly. Does not support snapshots, compression, or encryption natively. Consumed space equals the full virtual size unless the filesystem supports sparse files.

### qcow2 (QEMU Copy-On-Write v2)

The preferred format for KVM. Key features:

| Feature | Description |
|---------|-------------|
| **Copy-on-write** | Allocates clusters on first write; image grows on demand |
| **Backing files** | Overlay chains where a child image stores only deltas from a parent |
| **Internal snapshots** | Point-in-time snapshots stored within the same file |
| **Compression** | Per-cluster zlib compression (read-only clusters) |
| **Encryption** | AES or LUKS-based encryption of data clusters |
| **Preallocation modes** | `off` (sparse), `metadata` (metadata preallocated, data sparse), `falloc` (fallocate, fast), `full` (dd-style, slowest but guarantees space) |
| **lazy_refcounts** | Defers refcount updates for crash-unsafe but faster writes; run `qemu-img check -r all` after unclean shutdown |
| **Cluster size** | Default 64 KB; tunable at creation (larger clusters reduce metadata overhead but increase I/O amplification) |

## qemu-img Tool

`qemu-img` is the primary utility for managing disk images outside a running VM.

| Subcommand | Purpose |
|------------|---------|
| `create -f qcow2 [-b backing] [-o opts] file size` | Create a new image; `-o preallocation=metadata,lazy_refcounts=on` for tuned qcow2 |
| `info [--backing-chain] file` | Display format, virtual/actual size, snapshots; `--backing-chain` walks the full overlay chain |
| `convert -f src_fmt -O dst_fmt src dst` | Convert between formats; `-c` enables compression, `-p` shows progress |
| `resize file [+/-]size` | Grow or shrink the virtual size (guest must also resize its filesystem) |
| `snapshot -c tag file` | Create an internal snapshot |
| `snapshot -l file` | List internal snapshots |
| `snapshot -a tag file` | Apply (revert to) a snapshot |
| `snapshot -d tag file` | Delete a snapshot |
| `check [-r all] file` | Verify image consistency; `-r all` repairs refcount errors |
| `rebase -b new_backing file` | Change the backing file reference; `-u` for unsafe (metadata-only) rebase |
| `commit file` | Merge an overlay's data down into its backing file |
| `amend -o opts file` | Modify image options in place (e.g., enable lazy_refcounts) |
| `compare file1 file2` | Byte-compare two images; exit 0 if identical |
| `map file` | Show allocated/zero/data regions and their offsets |

## Storage Pool Types

### Directory Pool (default)

Stores images as files in a host directory. Simplest setup; default pool is `/var/lib/libvirt/images`.

```xml
<pool type='dir'>
  <name>default</name>
  <target>
    <path>/var/lib/libvirt/images</path>
  </target>
</pool>
```

### LVM Pool

Maps a volume group as a pool; logical volumes become storage volumes. Supports thin provisioning via LVM thin pools for space-efficient allocation.

```xml
<pool type='logical'>
  <name>lvm-pool</name>
  <source>
    <name>vg_vms</name>
  </source>
  <target>
    <path>/dev/vg_vms</path>
  </target>
</pool>
```

### NFS Pool

Mounts a remote NFS export. libvirt handles mounting/unmounting with pool start/stop.

```xml
<pool type='netfs'>
  <name>nfs-pool</name>
  <source>
    <host name='nfs-server'/>
    <dir path='/exports/vms'/>
    <format type='nfs'/>
  </source>
  <target>
    <path>/var/lib/libvirt/nfs-pool</path>
  </target>
</pool>
```

### iSCSI Pool

Connects to an iSCSI target. On the target side, `targetcli` configures LUNs and ACLs. On the initiator (KVM host), `iscsiadm` handles discovery and login. CHAP authentication secures the session.

```xml
<pool type='iscsi'>
  <name>iscsi-pool</name>
  <source>
    <host name='iscsi-target'/>
    <device path='iqn.2016-01.com.example:storage'/>
    <auth type='chap' username='kvmhost'>
      <secret type='iscsi' uuid='secret-uuid'/>
    </auth>
  </source>
</pool>
```

### Ceph RBD Pool

Accesses RADOS Block Device images in a Ceph cluster. Authentication uses CephX with a libvirt secret storing the client key.

```xml
<pool type='rbd'>
  <name>ceph-pool</name>
  <source>
    <host name='mon1' port='6789'/>
    <host name='mon2' port='6789'/>
    <name>libvirt-pool</name>
    <auth type='ceph' username='libvirt'>
      <secret uuid='ceph-secret-uuid'/>
    </auth>
  </source>
</pool>
```

Creating the libvirt secret:

```bash
virsh secret-define --file secret.xml
virsh secret-set-value --secret <uuid> --base64 $(ceph auth get-key client.libvirt)
```

### GlusterFS Pool

Uses the native QEMU GlusterFS driver (not FUSE mount) for better performance by communicating directly with glusterd.

```xml
<pool type='gluster'>
  <name>gluster-pool</name>
  <source>
    <host name='gluster-node'/>
    <dir path='/'/>
    <name>gv_vms</name>
  </source>
</pool>
```

## Pool and Volume Management

### Pool lifecycle commands

```bash
virsh pool-define-as <name> <type> [source-args] [target-path]
virsh pool-build <name>          # create underlying directory/VG/etc.
virsh pool-start <name>          # activate the pool
virsh pool-autostart <name>      # start on host boot
virsh pool-list --all            # list pools and state
virsh pool-refresh <name>        # rescan for new volumes
virsh pool-destroy <name>        # deactivate (does not delete data)
virsh pool-delete <name>         # remove underlying storage
virsh pool-undefine <name>       # remove pool definition
```

### Volume management commands

```bash
virsh vol-create-as <pool> <vol> <size> --format qcow2
virsh vol-list <pool>
virsh vol-info <vol> --pool <pool>
virsh vol-clone <src-vol> <dst-vol> --pool <pool>
virsh vol-resize <vol> <size> --pool <pool>
virsh vol-delete <vol> --pool <pool>
virsh vol-wipe <vol> --pool <pool>   # zero-fill before deletion
```

## Disk Configuration: virtio-blk vs virtio-scsi

Both paravirtual drivers are part of the [virtio-framework](virtio-framework.md). Their trade-offs:

| Aspect | virtio-blk | virtio-scsi |
|--------|-----------|-------------|
| Device naming in guest | `/dev/vdX` | `/dev/sdX` |
| Max devices per bus | ~28 (PCI slots) | Thousands (SCSI LUNs) |
| SCSI command passthrough | No | Yes |
| Discard/TRIM support | Limited | Full |
| Hotplug reliability | Basic | Robust |
| Multiqueue | Separate virtqueue per disk | Shared I/O queues across LUNs |
| Use case | Simple, fewer disks | Large-scale, feature-rich |

virtio-scsi is the recommended controller for production workloads requiring more than a handful of disks or SCSI features. See [kvm-performance-tuning](kvm-performance-tuning.md) for tuning guidance.

## Cache Modes

Cache modes control how the host handles guest I/O with respect to the page cache and disk flushes. Configured via `cache='mode'` on the `<driver>` element.

| Mode | Host page cache | O_DIRECT | Flushes (fsync) | Notes |
|------|:-:|:-:|:-:|-------|
| `none` | No | Yes | Honored | Safest for data; required for native AIO. Recommended default. |
| `writethrough` | Yes | No | Every write | Safe but slow; guest reads can hit host cache. |
| `writeback` | Yes | No | Honored | Better performance than writethrough; data at risk if host crashes before flush. |
| `directsync` | No | Yes | Every write | Like `none` + forced sync; maximum safety, lowest performance. |
| `unsafe` | Yes | No | Ignored | Fastest; all durability guarantees voided. Use only for ephemeral guests or installations. |

For the interaction between cache modes and the kernel [block-layer](block-layer.md), `cache=none` bypasses the host page cache entirely, reducing double-caching when the guest has its own cache.

## I/O Modes

The `io='mode'` attribute on `<driver>` selects the host-side I/O submission model:

- **threads** (default) -- QEMU dispatches I/O through a userspace thread pool. Compatible with all cache modes. Each I/O request is handled by a worker thread, which may introduce context-switch overhead under heavy load.
- **native** -- Uses Linux Asynchronous I/O (AIO / `io_submit`). Submits I/O directly to the kernel without extra threads, reducing latency and CPU overhead. **Requires `cache=none`** because Linux AIO mandates O_DIRECT.

For high-throughput workloads, `cache=none,io=native` is the standard recommendation. See [kvm-performance-tuning](kvm-performance-tuning.md).

## Disk Hot-Add and Hot-Remove

Disks can be attached or detached from a running guest without downtime using [libvirt-management](libvirt-management.md) commands:

```bash
# Attach a disk (virtio-scsi example)
virsh attach-disk <domain> /var/lib/libvirt/images/data.qcow2 sdb \
  --driver qemu --subdriver qcow2 --targetbus scsi --live

# Detach a disk
virsh detach-disk <domain> sdb --live
```

Adding `--config` alongside `--live` persists the change across reboots. The guest OS must support device hotplug (standard in modern Linux kernels). virtio-scsi handles hot-add/remove more reliably than virtio-blk due to its full SCSI layer emulation.

## See also

- [libvirt-management](libvirt-management.md)
- [virtio-framework](virtio-framework.md)
- [block-layer](block-layer.md)
- [kvm-performance-tuning](kvm-performance-tuning.md)
