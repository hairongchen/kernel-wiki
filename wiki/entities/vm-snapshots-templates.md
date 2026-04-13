---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [mastering-kvm-virtualization]
tags: [kvm, snapshots, templates, cloning, qcow2, libvirt]
---

# VM Snapshots, Cloning, and Templates

KVM/QEMU provides a rich set of mechanisms for capturing VM state, duplicating virtual machines, and building reusable images. These capabilities rely on the qcow2 disk format and are orchestrated through libvirt and its associated command-line tools.

## VM Templates

A **VM template** is a pre-configured, generalized disk image that serves as a golden master for deploying new VMs. Templates are not running machines; they are static base images from which clones are instantiated.

### Template Workflow

1. **Install** -- Create a VM and install the guest OS with the desired software stack.
2. **Configure** -- Apply all common settings: packages, security policies, network defaults.
3. **Generalize** -- Strip machine-specific identifiers so each clone gets unique identity.
4. **Mark as template** -- Set the image read-only or store it in a dedicated template directory.

### Generalization with virt-sysprep

`virt-sysprep` (from libguestfs) prepares a disk image for cloning by removing host-specific state. It operates on a **shut-down** guest image.

Key operations performed by default:

- Remove SSH host keys (`ssh-hostkeys`)
- Clear `/etc/machine-id` (`machine-id`)
- Reset hostname (`hostname`)
- Truncate log files (`logfiles`)
- Remove bash history (`bash-history`)
- Remove user credentials and random seeds

Use `--operations` to select specific operations:

```
virt-sysprep --operations ssh-hostkeys,machine-id,hostname -a template.qcow2
```

The `--firstboot` flag injects a script that runs once on the first boot of each clone, useful for setting a new hostname, registering with a configuration management system, or generating fresh SSH keys:

```
virt-sysprep --firstboot setup.sh -a template.qcow2
```

### virt-builder

`virt-builder` provides a faster alternative to manual template creation. It downloads pre-built OS images from a public or private repository and customizes them in one step:

```
virt-builder fedora-30 --size 20G --format qcow2 \
  --root-password password:changeme \
  --install vim,tmux \
  --firstboot-command 'hostnamectl set-hostname newvm'
```

This avoids a full installation cycle and is well-suited for CI/CD and lab environments.

## VM Cloning

Cloning creates a new VM from an existing one (or template) with a distinct identity -- new UUID, new MAC addresses, and a separate disk image.

### virt-clone

The primary tool for cloning is `virt-clone`, part of the virt-manager package:

```
virt-clone --original template-vm --name clone-vm --auto-clone
```

Key flags:

| Flag | Purpose |
|------|---------|
| `--original` | Name of the source VM (must be shut down) |
| `--name` | Name for the new VM |
| `--auto-clone` | Automatically generate new disk paths and MAC addresses |
| `--file` | Explicit path for the clone's disk image |

`virt-clone` generates a new UUID and MAC address for each cloned NIC, preventing identity collisions on the network.

### Full Clone vs Linked (Thin) Clone

A **full clone** copies the entire disk image. It is self-contained but consumes the full disk footprint.

A **linked clone** (also called a thin clone) uses a **backing file** mechanism in qcow2. The clone's overlay stores only the differences from the base:

```
qemu-img create -f qcow2 -b /var/lib/libvirt/images/base.qcow2 \
  -F qcow2 /var/lib/libvirt/images/linked-clone.qcow2
```

Linked clones are space-efficient and fast to create, but depend on the backing file remaining accessible and unmodified. They are ideal for ephemeral workloads and development environments.

## Internal Snapshots

Internal snapshots store all snapshot data **within a single qcow2 file**. The qcow2 format uses a reference-count mechanism and copy-on-write to keep multiple point-in-time states in the same file.

### Types

- **Disk-only snapshot** -- Captures disk state only. The VM can be running or stopped.
- **System checkpoint** -- Captures disk state **and** memory/device state. Allows restoring a VM to the exact running state, including in-flight processes.

### Management Commands

Create a snapshot:

```
virsh snapshot-create-as domain snap1 --description "Before upgrade"
```

For a system checkpoint (includes memory), the VM must be running and the snapshot is created live.

List snapshots:

```
virsh snapshot-list domain
virsh snapshot-list domain --tree    # hierarchical view
```

Inspect a snapshot:

```
virsh snapshot-info domain snap1
```

Revert to a snapshot:

```
virsh snapshot-revert domain snap1
```

Delete a snapshot:

```
virsh snapshot-delete domain snap1
```

### Limitations

- **Performance** -- The qcow2 file grows with each snapshot and read performance degrades as the internal reference chain lengthens, particularly on large disks.
- **Format lock-in** -- Internal snapshots are exclusive to qcow2; raw images do not support them.
- **Manageability** -- All state is bundled in one file, making it harder to manage backup and replication of individual snapshots.

## External Snapshots

External snapshots create a **new overlay qcow2 file** for each snapshot. The original image becomes a read-only backing file, and all new writes go to the overlay.

### Overlay Chain

Each successive snapshot extends the chain:

```
base.qcow2 <-- snap1.qcow2 <-- snap2.qcow2 <-- active.qcow2
(read-only)    (read-only)      (read-only)      (read-write)
```

Reads traverse the chain from the active layer downward until the requested block is found.

### Creating External Snapshots

```
virsh snapshot-create-as domain snap-ext \
  --disk-only \
  --diskspec vda,snapshot=external,file=/var/lib/libvirt/images/snap-ext.qcow2
```

The `--disk-only` flag signals an external snapshot. The `--diskspec` option specifies the target disk, snapshot mode, and the file path for the overlay.

### Inspecting the Chain

```
qemu-img info --backing-chain /var/lib/libvirt/images/active.qcow2
```

This displays each layer in the chain with its backing file reference, format, and virtual/actual size.

### Advantages over Internal Snapshots

- Better I/O performance (thinner overlay files).
- Each layer is a separate file, simplifying backup and transfer.
- Preferred for production use and integration with external backup tools.

## Blockcommit

**Blockcommit** merges data from an overlay **downward** into its backing file, shortening the chain. It is the primary tool for flattening external snapshot chains.

### Active Layer Commit

To merge the active (topmost) overlay into its backing file and pivot the VM to use the merged result:

```
virsh blockcommit domain vda --active --verbose --pivot
```

The `--pivot` flag switches the live VM to use the committed base image once the merge completes. The `--active` flag is required when committing the currently active layer.

### Intermediate Layer Commit

For non-active (intermediate) layers, `--active` and `--pivot` are not needed:

```
virsh blockcommit domain vda --base snap1.qcow2 --top snap2.qcow2 --verbose
```

This merges snap2 into snap1, removing one link from the chain.

## Blockpull

**Blockpull** is the inverse of blockcommit: it pulls data from backing files **upward** into the active overlay, making the active image self-contained.

### Full Flattening

```
virsh blockpull domain vda --verbose --wait
```

This copies all data from every backing file into the active overlay. Once complete, the active image has no backing file dependency and the chain is fully flattened.

### Partial Flattening

Use `--base` to pull only down to a specific point in the chain:

```
virsh blockpull domain vda --base base.qcow2 --verbose --wait
```

This merges all layers above `base.qcow2` into the active overlay, but retains the dependency on `base.qcow2`.

### Blockcommit vs Blockpull

| Aspect | Blockcommit | Blockpull |
|--------|-------------|-----------|
| Direction | Merges overlay into backing file (downward) | Pulls backing data into active overlay (upward) |
| Result | Backing file grows, overlay removed | Active overlay grows, backing files can be removed |
| Use case | Collapse intermediate layers | Make active image self-contained |

## Best Practices

- **Keep snapshot chains short.** Long chains degrade read performance because each read may traverse multiple layers. Periodically flatten with blockcommit or blockpull.
- **Prefer external snapshots for production.** They offer better performance, granular backup, and cleaner management than internal snapshots.
- **Verify image integrity.** Run `qemu-img check <image>` after snapshot operations to detect corruption in the reference counts or backing chain.
- **Back up both overlay and backing files.** An overlay is useless without its backing file. Ensure backup procedures capture the entire chain.
- **Use templates with virt-sysprep for consistent deployments.** Generalized images prevent identity conflicts and reduce provisioning time.
- **Protect backing files.** Mark template and backing images as read-only at the filesystem level to prevent accidental modification.
- **Document your chains.** For complex environments, track which overlays depend on which backing files to avoid orphaning layers during cleanup.

## See also

- [libvirt-management](libvirt-management.md)
- [kvm-storage](kvm-storage.md)
- [vm-live-migration](vm-live-migration.md)
