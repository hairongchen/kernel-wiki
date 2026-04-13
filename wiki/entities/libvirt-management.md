---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [mastering-kvm-virtualization]
tags: [libvirt, virsh, virt-manager, kvm, management]
---

# libvirt Management Layer

libvirt is the standard open-source management layer for KVM virtualization. It provides a stable, cross-hypervisor API and a set of tools for creating, configuring, monitoring, and controlling virtual machines. This page covers libvirt's architecture, its primary CLI and GUI tools, VM definition mechanics, and the broader ecosystem of utilities described in *Mastering KVM Virtualization* (Chirammal et al., Packt, 2016).

## libvirt Architecture

### The libvirtd Daemon

The `libvirtd` daemon is the central management process. It runs on the host, accepts connections from clients (local or remote), and translates high-level API calls into hypervisor-specific operations — for KVM, that means interacting with QEMU processes and the kernel's KVM module. The daemon manages VM lifecycle, device hotplug, networking, storage, and more.

Key characteristics:

- Runs as root (for `qemu:///system`) or per-user (for `qemu:///session`).
- Maintains persistent VM definitions in `/etc/libvirt/qemu/`.
- Tracks running VM state, including QEMU process PIDs and monitor sockets.
- Provides event-driven notifications for state changes.

### Connection URIs

libvirt uses URIs to identify which hypervisor and host to connect to:

| URI | Description |
|-----|-------------|
| `qemu:///system` | Local system-level QEMU/KVM (root privileges, full networking) |
| `qemu:///session` | Local user-session QEMU/KVM (limited networking, no bridging) |
| `qemu+ssh://host/system` | Remote host over SSH tunnel |
| `qemu+tcp://host/system` | Remote host over unencrypted TCP (not recommended) |
| `qemu+tls://host/system` | Remote host over TLS-encrypted TCP |

The `system` URI is the standard choice for production environments because it grants access to host-level networking (bridges, macvtap) and privileged device assignment. See [kvm-networking](kvm-networking.md) for networking details.

### API Levels

libvirt exposes its functionality through several layers:

- **C API (`libvirt.h`)** — the canonical, low-level interface. All other bindings wrap this.
- **Language bindings** — Python (`libvirt-python`), Go, Perl, Ruby, Java, and others.
- **virsh** — the official command-line client (see below).
- **virt-manager** — the official GUI client.
- **HTTP/REST** — via `oVirt` or other higher-level platforms that consume the libvirt API.

## virsh CLI

`virsh` is the primary command-line tool for managing VMs through libvirt. Commands can be run as one-shot (`virsh <command>`) or in an interactive shell (`virsh` then type commands).

### Domain Lifecycle

| Command | Purpose |
|---------|---------|
| `virsh list --all` | List all domains (running and inactive) |
| `virsh start <domain>` | Boot a defined (inactive) domain |
| `virsh shutdown <domain>` | Request graceful ACPI shutdown |
| `virsh destroy <domain>` | Force-stop immediately (like pulling the power cord) |
| `virsh suspend <domain>` | Pause execution (freeze vCPUs) |
| `virsh resume <domain>` | Unpause a suspended domain |
| `virsh reboot <domain>` | Request graceful ACPI reboot |
| `virsh reset <domain>` | Hard reset (no graceful shutdown) |
| `virsh save <domain> <file>` | Save running state to file (like hibernate) |
| `virsh restore <file>` | Restore a previously saved domain |
| `virsh managedsave <domain>` | Save state and auto-restore on next start |

### Domain Information

| Command | Purpose |
|---------|---------|
| `virsh dominfo <domain>` | Summary: state, memory, vCPUs, autostart, UUID |
| `virsh domstats <domain>` | Detailed statistics (CPU, balloon, block, net) |
| `virsh dumpxml <domain>` | Dump full XML definition to stdout |
| `virsh edit <domain>` | Edit XML definition in `$EDITOR` (validates on save) |
| `virsh domblklist <domain>` | List block devices |
| `virsh domiflist <domain>` | List network interfaces |
| `virsh vcpuinfo <domain>` | vCPU-to-pCPU mapping and CPU time |

### Domain Configuration

| Command | Purpose |
|---------|---------|
| `virsh setvcpus <domain> <count>` | Change active vCPU count (hot-plug if supported) |
| `virsh setmem <domain> <KiB>` | Adjust memory via balloon driver |
| `virsh vcpucount <domain>` | Show current/max, live/config vCPU counts |
| `virsh vcpupin <domain> <vcpu> <cpulist>` | Pin a vCPU to specific host CPUs |
| `virsh autostart <domain>` | Enable auto-start on host boot |
| `virsh autostart --disable <domain>` | Disable auto-start |

For performance tuning of vCPU pinning and memory, see [kvm-performance-tuning](kvm-performance-tuning.md).

### Snapshots and Migration

| Command | Purpose |
|---------|---------|
| `virsh snapshot-create-as <domain> <name>` | Create a named snapshot |
| `virsh snapshot-list <domain>` | List snapshots |
| `virsh snapshot-revert <domain> <name>` | Revert to a snapshot |
| `virsh migrate <domain> <dest-uri>` | Live-migrate a domain |

For live migration details, see [vm-live-migration](vm-live-migration.md).

## VM Definition (XML Format)

libvirt defines virtual machines using XML documents. The key top-level sections are:

```xml
<domain type='kvm'>
  <name>myvm</name>
  <uuid>auto-generated</uuid>
  <memory unit='GiB'>4</memory>
  <currentMemory unit='GiB'>4</currentMemory>
  <vcpu placement='static'>2</vcpu>
  <os>
    <type arch='x86_64' machine='pc-q35-6.2'>hvm</type>
    <boot dev='hd'/>
  </os>
  <features>
    <acpi/><apic/>
  </features>
  <devices>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/myvm.qcow2'/>
      <target dev='vda' bus='virtio'/>
    </disk>
    <interface type='network'>
      <source network='default'/>
      <model type='virtio'/>
    </interface>
    <graphics type='vnc' port='-1' autoport='yes'/>
    <channel type='unix'>
      <target type='virtio' name='org.qemu.guest_agent.0'/>
    </channel>
  </devices>
</domain>
```

### Storage and Lifecycle of Definitions

- **Persistent definitions** are stored as XML files in `/etc/libvirt/qemu/`.
- `virsh define <file.xml>` — register a persistent VM (does not start it).
- `virsh create <file.xml>` — create and immediately start a transient VM (gone when stopped).
- `virsh undefine <domain>` — remove the persistent definition. Add `--remove-all-storage` to also delete disk images.
- `virsh undefine --nvram <domain>` — required for UEFI guests to also remove NVRAM files.

For storage pool and volume management, see [kvm-storage](kvm-storage.md).

## virt-install

`virt-install` automates VM creation by generating the XML definition and optionally launching an installation. Key parameters:

| Parameter | Purpose |
|-----------|---------|
| `--name` | VM name (must be unique) |
| `--memory` | RAM in MiB (e.g., `2048` or `memory=2048,maxmemory=4096`) |
| `--vcpus` | vCPU count (e.g., `2` or `vcpus=2,maxvcpus=4`) |
| `--disk` | Disk specification (e.g., `path=/var/lib/libvirt/images/vm.qcow2,size=20,format=qcow2`) |
| `--cdrom` | ISO image path for CD-ROM-based install |
| `--location` | URL or path to install tree (enables text-mode/kickstart installs) |
| `--os-variant` | Guest OS type for optimized defaults (e.g., `rhel8.0`, `ubuntu20.04`). List with `osinfo-query os` |
| `--network` | Network config (e.g., `network=default,model=virtio` or `bridge=br0`) |
| `--graphics` | Display type (`vnc`, `spice`, `none` for headless) |
| `--boot` | Boot options (e.g., `uefi`, `hd`, `cdrom`, `network`) |
| `--import` | Skip installation; boot from existing disk image |
| `--pxe` | Boot from PXE network install |
| `--extra-args` | Kernel command-line arguments (works with `--location`) |
| `--initrd-inject` | Inject a kickstart/preseed file into the initrd |

Example creating a VM from an ISO:

```bash
virt-install \
  --name centos8 \
  --memory 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/centos8.qcow2,size=20,format=qcow2 \
  --cdrom /path/to/CentOS-8-x86_64-dvd.iso \
  --os-variant centos8 \
  --network network=default,model=virtio \
  --graphics vnc
```

## virt-manager

`virt-manager` is the GTK-based GUI for libvirt. It provides:

- **VM creation wizard** — graphical equivalent of `virt-install`, with guided OS selection and resource allocation.
- **Console access** — integrated VNC/SPICE viewer for guest displays.
- **Hardware management** — add/remove/configure virtual devices (disks, NICs, USB, PCI passthrough) through a details panel.
- **Performance graphs** — real-time CPU, memory, disk I/O, and network charts per VM.
- **Remote host management** — connect to remote libvirtd instances via SSH, TLS, or TCP.
- **Snapshot management** — create, revert, and delete snapshots through the UI.

For headless servers, `virt-manager` can be used from a workstation by connecting to the remote `qemu+ssh://` URI.

## QMP / QEMU Monitor

The QEMU Machine Protocol (QMP) is the JSON-based interface between libvirt and each QEMU process. For advanced troubleshooting or features not yet exposed by libvirt, `virsh` can pass commands directly to the QEMU monitor:

```bash
# HMP (Human Monitor Protocol) — text-based, easier for interactive use
virsh qemu-monitor-command <domain> --hmp "info cpus"
virsh qemu-monitor-command <domain> --hmp "info block"
virsh qemu-monitor-command <domain> --hmp "info network"
virsh qemu-monitor-command <domain> --hmp "info migrate"
# QMP (JSON) — structured output, better for scripting
virsh qemu-monitor-command <domain> '{"execute":"query-block"}'
```

Useful for inspecting internal QEMU state not exposed by libvirt. See [qemu-kvm-overview](qemu-kvm-overview.md) for more on QEMU internals.

## Monitoring and Logging

### Runtime Monitoring

- **`virt-top`** — a `top`-like utility for VMs. Shows per-domain CPU, memory, block I/O, and network statistics in real time.
- **`perf kvm stat`** — kernel perf tool integration for KVM. Shows VM exit reasons and frequency, useful for diagnosing performance problems (e.g., excessive `HLT` or `EPT_VIOLATION` exits).
- **`virsh domstats`** — programmatic access to domain statistics (CPU time, balloon info, block/net counters).

### QEMU Logs

Each VM's QEMU process logs to `/var/log/libvirt/qemu/<domain>.log`, capturing QEMU command-line arguments, device initialization, errors, and shutdown/crash information.

### libvirtd Logging Configuration

Logging is configured in `/etc/libvirt/libvirtd.conf`:

```
log_level = 1          # 1=DEBUG, 2=INFO, 3=WARN, 4=ERROR
log_filters = "1:qemu 3:event"   # per-module filtering
log_outputs = "1:file:/var/log/libvirt/libvirtd.log"
```

For runtime log changes without restarting libvirtd, use `virt-admin`:

```bash
virt-admin daemon-log-filters "1:qemu 1:util.json"
virt-admin daemon-log-outputs "1:file:/var/log/libvirt/libvirtd.log"
```

## libguestfs Tools

libguestfs provides a suite of tools for manipulating VM disk images without booting the guest.

### virt-sysprep

Generalizes a VM image for cloning by removing machine-specific data:

```bash
virt-sysprep -d <domain>        # operate on a libvirt domain (must be shut down)
virt-sysprep -a disk.qcow2      # operate directly on an image file
```

Removes: SSH host keys, persistent net rules, machine ID, user accounts (optional), log files, and hostname/IP configuration.

### virt-builder

Rapidly creates new VM images from pre-built OS templates:

```bash
virt-builder fedora-33 \
  --size 20G \
  --format qcow2 \
  --root-password password:changeme \
  -o /var/lib/libvirt/images/fedora33.qcow2
```

Downloads a minimal cloud image and customizes it in one step.

### virt-customize

Applies modifications to an existing disk image:

```bash
virt-customize -a disk.qcow2 \
  --install nginx,vim \
  --run-command 'systemctl enable nginx' \
  --ssh-inject root:file:/root/.ssh/id_rsa.pub
```

### guestfish

Interactive shell for filesystem-level access to VM images. Useful for rescue operations, configuration changes, and forensic inspection without booting the guest:

```bash
guestfish --rw -a disk.qcow2 -i   # mount and inspect automatically
```

## See also

- [qemu-kvm-overview](qemu-kvm-overview.md)
- [kvm-networking](kvm-networking.md)
- [kvm-storage](kvm-storage.md)
- [vm-live-migration](vm-live-migration.md)
- [kvm-performance-tuning](kvm-performance-tuning.md)
