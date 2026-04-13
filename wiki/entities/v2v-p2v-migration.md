---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [mastering-kvm-virtualization]
tags: [kvm, v2v, p2v, migration, virt-v2v, virt-p2v, conversion]
---

# V2V and P2V Migration to KVM

Virtual-to-virtual (V2V) and physical-to-virtual (P2V) migration convert existing workloads into KVM-compatible virtual machines managed by [libvirt-management](libvirt-management.md). The primary tools are **virt-v2v** (for V2V) and **virt-p2v** (for P2V), both part of the libguestfs ecosystem.

## virt-v2v Overview

`virt-v2v` converts virtual machines from foreign hypervisors -- VMware ESXi/vSphere, Citrix Xen, and Microsoft Hyper-V -- into KVM-compatible libvirt domains. It handles the full conversion pipeline: disk format translation, driver replacement, boot configuration adjustment, and libvirt XML generation. The tool operates on powered-off guests and produces a ready-to-run KVM domain.

A typical invocation looks like:

```
virt-v2v -ic vpx://vcenter.example.com/Datacenter/esxi-host?no_verify=1 \
    -o local -os /var/tmp/output \
    --bridge br0 \
    "guest-name"
```

## Input Modes

Input modes specify where virt-v2v reads the source VM from.

| Flag | Source | Description |
|------|--------|-------------|
| `-i vmx` | VMware VMX file | Reads a `.vmx` configuration file and its associated VMDK disk images directly from a local or NFS-mounted datastore. |
| `-ic vpx://` | vCenter / ESXi (HTTPS) | Connects to a VMware vCenter Server or standalone ESXi host via the VMware SDK over HTTPS. Requires valid credentials. |
| `-i ova` | OVA archive | Reads an OVA (Open Virtualization Appliance) archive, which bundles an OVF descriptor and one or more VMDK disks into a single tar file. |
| `-ic xen+ssh://` | Xen via SSH | Connects to a Xen hypervisor over SSH, reading the guest configuration and disk images remotely. |
| `-i disk` | Raw disk image | Converts a standalone disk image (raw, qcow2, VMDK, etc.) without any VM configuration metadata. Useful for bare disk imports. |

When using `-ic vpx://`, the URI follows the pattern:

```
vpx://user@vcenter-host/Datacenter/esxi-host?no_verify=1
```

The `no_verify=1` parameter disables SSL certificate verification, which is common in lab environments. For production, proper certificate handling is recommended.

## Output Modes

Output modes control where the converted VM is written.

| Flag | Target | Description |
|------|--------|-------------|
| `-o local` | Local directory | Writes the converted disk image and a libvirt XML definition to a directory specified by `-os /path`. Good for inspection before import. |
| `-o libvirt` | Direct import | Imports the converted VM directly into the local libvirt instance, creating the domain and placing disks in the default [kvm-storage](kvm-storage.md) pool. |
| `-o rhev` | oVirt/RHEV export domain | Writes the converted VM to an oVirt/RHEV export storage domain (NFS-mounted), from which it can be imported through the oVirt management interface. |
| `-o glance` | OpenStack Glance | Uploads the converted disk image to an OpenStack Glance image service, making it available for launching instances in OpenStack. |

## Conversion Steps

virt-v2v performs a multi-stage conversion pipeline:

### 1. Disk Format Conversion

Source disk images (VMDK, VHD, raw) are converted to **qcow2** format, which is the standard format for KVM. This provides thin provisioning, snapshots, and efficient storage. The conversion is handled by `qemu-img convert` internally.

### 2. Hypervisor Driver Removal

Foreign hypervisor-specific drivers and agents are removed from the guest:

- **VMware Tools** -- paravirtual drivers (vmxnet3, pvscsi), the VMware Tools agent, and related kernel modules
- **Xen PV drivers** -- paravirtual block and network drivers, Xen-specific kernel configurations
- **Hyper-V Integration Services** -- synthetic drivers and agents

### 3. VirtIO Driver Installation

KVM-optimized **virtio** paravirtual drivers are installed to replace the removed hypervisor drivers:

- **virtio-blk** -- paravirtual block device driver for high-performance disk I/O
- **virtio-net** -- paravirtual network driver
- **virtio-balloon** -- memory ballooning driver for dynamic memory management

For Linux guests, the appropriate kernel modules are added to the initramfs/initrd. For Windows guests, virt-v2v injects the virtio drivers from a driver ISO.

### 4. Boot Configuration Adjustment

The guest boot configuration is updated to reflect the new virtual hardware:

- **GRUB** configuration is modified to reference virtio devices (e.g., `/dev/vda` instead of `/dev/sda`)
- **/etc/fstab** entries are updated if device naming changes
- Initramfs is regenerated to include virtio modules
- **UEFI vs BIOS**: virt-v2v detects whether the source VM uses UEFI or BIOS firmware and configures the KVM domain accordingly, selecting the appropriate OVMF firmware for UEFI guests

### 5. Libvirt XML Generation

A libvirt domain XML definition is generated for the converted VM, specifying:

- CPU model and topology
- Memory allocation
- Disk devices using virtio bus
- Network interfaces mapped to the target bridge or network
- Display (SPICE/VNC) and other peripheral devices

## Key Flags

| Flag | Purpose |
|------|---------|
| `--bridge <name>` | Map all guest network interfaces to a specific host bridge (e.g., `br0`). |
| `--network <name>` | Map all guest NICs to a named libvirt virtual network. |
| `-v` / `--verbose` | Enable verbose output for debugging conversion issues. |
| `--machine-readable` | Produce machine-parseable output for scripting and automation pipelines. |

## virt-p2v: Physical-to-Virtual Conversion

`virt-p2v` converts a physical machine into a KVM virtual machine. It uses a **two-tier architecture** that separates the data capture (on the physical host) from the conversion logic (on a remote server).

### Architecture

```
 Physical Machine                Conversion Server
 ┌──────────────┐    SSH/NBD     ┌──────────────────┐
 │  virt-p2v    │ ────────────►  │  virt-v2v         │
 │  (client)    │   disk data    │  (server-side)    │
 │              │                │                    │
 │  Bootable    │                │  Performs full     │
 │  ISO/USB     │                │  conversion        │
 └──────────────┘                └──────────────────┘
```

- **P2V client**: A minimal bootable environment (ISO or USB) running `virt-p2v`, which captures the physical machine's disk data and streams it to the conversion server.
- **Conversion server**: A Linux host running `virt-v2v`, which receives the disk data and performs the standard conversion pipeline (driver swap, boot config, XML generation).

### Workflow

1. **Create boot media**: Use `virt-p2v-make-disk` (or `virt-p2v-make-kickstart`) to build a bootable ISO or USB image containing the virt-p2v client.

   ```
   virt-p2v-make-disk -o /var/tmp/p2v-boot.iso
   ```

2. **Boot the physical machine**: Boot the target physical machine from the P2V media (CD/USB). The machine loads a minimal Linux environment.

3. **Configure via GUI**: The virt-p2v GUI launches, prompting the operator to:
   - Enter the conversion server hostname/IP and SSH credentials
   - Select which disks and network interfaces to convert
   - Set the target VM name and resource allocation

4. **Stream disk data**: virt-p2v streams the physical disks to the conversion server over SSH using NBD (Network Block Device). The conversion server runs virt-v2v on the received data.

5. **Finalize**: The conversion server produces a KVM-ready VM with virtio drivers, updated boot config, and libvirt XML.

## Considerations

### VMware Source Specifics

- **Credentials**: vCenter/ESXi connections require valid administrator credentials. Passwords can be provided interactively or via environment variables.
- **SSL certificates**: Production environments should configure proper CA certificates. The `?no_verify=1` URI parameter is a workaround for self-signed certificates.
- **Guest power state**: The source VM **must be powered off** before conversion. virt-v2v will refuse to convert a running guest.

### P2V Considerations

- **Network bandwidth**: P2V transfers entire physical disks over the network. For large disks, this can take hours. A dedicated high-speed network link between the physical machine and the conversion server is recommended.
- **Hardware differences**: Physical machines may have hardware-specific drivers or configurations (RAID controllers, custom network cards) that require attention post-conversion.

### General Best Practices

- **Test before decommissioning**: Always boot and validate the converted VM before decommissioning the original source (physical machine or foreign VM). Verify application functionality, network connectivity, and storage integrity.
- **Network mapping**: Plan bridge and network mappings in advance using `--bridge` or `--network` to ensure the converted VM lands on the correct virtual network segment.
- **Storage planning**: Ensure the target storage pool or export domain has sufficient space for the converted disk images. See [kvm-storage](kvm-storage.md) for storage pool configuration.
- **Live migration post-conversion**: Once a VM is successfully converted and running under KVM, it can participate in [vm-live-migration](vm-live-migration.md) like any native KVM guest.

## See also

- [libvirt-management](libvirt-management.md)
- [kvm-storage](kvm-storage.md)
- [vm-live-migration](vm-live-migration.md)
- [qemu-kvm-overview](qemu-kvm-overview.md)
- [virtio-framework](virtio-framework.md)
