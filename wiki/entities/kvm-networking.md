---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [mastering-kvm-virtualization]
tags: [kvm, networking, ovs, sr-iov, macvtap, bridge, libvirt]
---

# KVM Virtual Networking

KVM virtual networking connects guest VMs to each other and to external networks using a combination of kernel-level bridging, userspace daemons, and hardware offload mechanisms. The networking stack spans multiple layers: Linux bridge or Open vSwitch at L2, tap/macvtap devices as VM-facing endpoints, [virtio-framework](virtio-framework.md) for paravirtualized I/O, and SR-IOV for direct hardware passthrough. [libvirt-management](libvirt-management.md) provides a unified XML-based configuration interface across all these backends.

## Linux Bridge Networking

A Linux bridge operates as a software L2 switch inside the host kernel, forwarding Ethernet frames between attached interfaces based on MAC address learning. Each VM connects to the bridge through a **tap device** -- a virtual network interface that presents as a character device (`/dev/tapN`) to the QEMU process and as an Ethernet port to the kernel networking stack.

### Bridge setup

```bash
# brctl (bridge-utils)                  # iproute2 equivalent
brctl addbr br0                         # ip link add name br0 type bridge
brctl addif br0 eth0                    # ip link set eth0 master br0
brctl stp br0 on                        # ip link set br0 type bridge stp_state 1
brctl show                              # bridge link show
```

When QEMU starts a VM with `-netdev tap,id=net0,ifname=vnet0`, the host kernel creates `vnet0` as a tap device. The bridge forwards frames between `vnet0` and other ports (physical NICs or other VM tap devices) exactly as a hardware switch would. The [bridging](bridging.md) subsystem in the kernel handles MAC learning, STP, and frame forwarding at the [net-device](net-device.md) layer.

## libvirt Network Types

libvirt abstracts network configuration into named network objects managed via `virsh net-*` commands.

### NAT (default)

The default network (`virbr0`) provides NAT-based connectivity. libvirt launches a **dnsmasq** process bound to the bridge to serve DHCP and DNS to guests. Outbound traffic is masqueraded via iptables.

```xml
<network>
  <name>default</name>
  <bridge name="virbr0"/>
  <forward mode="nat"/>
  <ip address="192.168.122.1" netmask="255.255.255.0">
    <dhcp>
      <range start="192.168.122.2" end="192.168.122.254"/>
    </dhcp>
  </ip>
</network>
```

### Routed

Guests receive routable IPs; the host acts as a router. No NAT is applied, but iptables rules allow forwarding. External routers must have routes pointing to the host for the guest subnet.

```xml
<forward mode="route"/>
```

### Isolated

Like NAT but with no `<forward>` element, so guests can communicate with each other and the host but have no external connectivity.

### Bridged (existing bridge)

Delegates to a pre-existing Linux bridge or OVS bridge. libvirt attaches tap devices directly without creating its own bridge.

```xml
<forward mode="bridge"/>
<bridge name="br0"/>
```

### Key virsh commands

| Command | Purpose |
|---------|---------|
| `virsh net-list --all` | list all defined networks |
| `virsh net-define net.xml` | define a persistent network from XML |
| `virsh net-start default` | activate a network |
| `virsh net-autostart default` | mark network for auto-start at boot |
| `virsh net-dumpxml default` | display running XML configuration |
| `virsh net-destroy default` | stop a running network |

## Open vSwitch (OVS)

OVS is a production-grade multilayer virtual switch supporting standard management interfaces (NetFlow, sFlow, LACP, 802.1Q VLANs) and centralized SDN control via OpenFlow.

### Architecture

- **ovs-vswitchd**: userspace daemon implementing the switch logic and OpenFlow protocol handling.
- **ovsdb-server**: database server storing switch configuration (bridges, ports, interfaces) in OVSDB format.
- **Kernel datapath module** (`openvswitch.ko`): performs fast-path packet forwarding in the kernel. On a cache miss, packets are sent to `ovs-vswitchd` for flow lookup, and the resulting action is cached in the kernel for subsequent packets.

### VLANs

```bash
# Access port -- tags untagged traffic with VLAN 100
ovs-vsctl add-port br0 vnet0 tag=100

# Trunk port -- carries VLANs 100, 200, 300
ovs-vsctl add-port br0 eth1 trunks=100,200,300
```

### GRE and VXLAN tunnels

OVS supports overlay tunnels for connecting bridges across hosts, enabling VM mobility without L2 adjacency between hypervisors.

```bash
ovs-vsctl add-port br0 gre0 -- set interface gre0 type=gre options:remote_ip=10.0.0.2
ovs-vsctl add-port br0 vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=10.0.0.2 options:key=5000
```

### OpenFlow rules

```bash
ovs-ofctl dump-flows br0                                        # dump current flows
ovs-ofctl add-flow br0 "in_port=1,actions=output:2"             # forward port 1 -> port 2
ovs-ofctl add-flow br0 "dl_src=00:11:22:33:44:55,actions=drop"  # drop by MAC
```

### Port mirroring

```bash
ovs-vsctl -- set bridge br0 mirrors=@m \
  -- --id=@src get port vnet0 \
  -- --id=@dst get port vnet1 \
  -- --id=@m create mirror name=mirror0 select-src-port=@src output-port=@dst
```

### libvirt integration

To attach a VM to an OVS bridge, use `<virtualport type='openvswitch'/>` in the domain XML:

```xml
<interface type='bridge'>
  <source bridge='ovsbr0'/>
  <virtualport type='openvswitch'>
    <parameters profileid='myprofile'/>
  </virtualport>
  <model type='virtio'/>
</interface>
```

## macvtap and macvlan

macvtap combines macvlan (MAC-based virtual LAN sub-interfaces on a physical NIC) with the tap device interface consumed by QEMU. This eliminates the need for a bridge entirely -- VMs connect almost directly to the physical network.

### Four modes

| Mode | Behavior |
|------|----------|
| **VEPA** | All frames, including VM-to-VM on the same host, are sent out the physical NIC to an external IEEE 802.1Qbg-capable switch, which reflects them back. Enables centralized switching and policy enforcement. |
| **Bridge** | The macvlan driver switches frames between sub-interfaces locally, acting as a simple bridge. VM-to-VM traffic on the same host does not hit the physical NIC. |
| **Private** | Like VEPA, but VMs on the same host are completely isolated from each other even if the external switch reflects frames back. |
| **Passthrough** | Gives a single VM exclusive access to the physical NIC. The NIC's MAC is replaced with the VM's MAC. Useful for SR-IOV VFs. |

### XML configuration

```xml
<interface type='direct'>
  <source dev='eth0' mode='bridge'/>
  <model type='virtio'/>
</interface>
```

### Host-VM communication limitation

A significant constraint of macvtap is that the **host cannot communicate with its own VMs** through the macvtap interface. Traffic from the host's IP stack and VM traffic are isolated at the macvlan layer. Workarounds include adding a separate host-only bridge interface or using an additional NIC.

## SR-IOV (Single Root I/O Virtualization)

SR-IOV allows a single physical PCIe device (the **Physical Function**, PF) to present multiple lightweight **Virtual Functions** (VFs) that can be directly assigned to VMs, bypassing the hypervisor data path entirely. This provides near-native network performance.

### Requirements

- SR-IOV capable NIC (e.g., Intel 82599, Mellanox ConnectX)
- IOMMU enabled: `intel_iommu=on` (Intel) or `amd_iommu=on` (AMD) in kernel boot parameters
- VT-d / AMD-Vi support in the BIOS

### Enabling VFs

```bash
cat /sys/class/net/eth0/device/sriov_totalvfs            # check max VFs supported
echo 4 > /sys/class/net/eth0/device/sriov_numvfs         # create 4 VFs
lspci | grep "Virtual Function"                          # verify
```

### Assigning VFs to VMs

**Method 1: hostdev (generic PCI passthrough)**

```xml
<hostdev mode='subsystem' type='pci' managed='yes'>
  <source>
    <address domain='0x0000' bus='0x03' slot='0x10' function='0x0'/>
  </source>
</hostdev>
```

**Method 2: interface type='hostdev' (network-aware passthrough)**

This method lets libvirt set MAC address and VLAN for the VF before passing it to the guest:

```xml
<interface type='hostdev' managed='yes'>
  <source>
    <address type='pci' domain='0x0000' bus='0x03' slot='0x10' function='0x0'/>
  </source>
  <mac address='52:54:00:12:34:56'/>
  <vlan>
    <tag id='100'/>
  </vlan>
</interface>
```

### libvirt nodedev commands

```bash
virsh nodedev-list --cap pci          # list PCI devices
virsh nodedev-dumpxml pci_0000_03_10_0  # show device details
virsh nodedev-detach pci_0000_03_10_0   # detach from host driver
virsh nodedev-reattach pci_0000_03_10_0 # reattach to host driver
```

## Network Filtering (nwfilter)

libvirt's nwfilter framework provides firewall rules at the VM interface level using **ebtables** (L2 filtering) and **iptables** (L3/L4 filtering). Filters are defined as reusable XML objects and can be chained.

### Predefined filters

| Filter | Purpose |
|--------|---------|
| `clean-traffic` | Umbrella filter combining anti-spoofing rules (prevents IP, MAC, ARP spoofing) |
| `no-mac-spoofing` | Blocks frames with a source MAC not matching the VM's assigned MAC |
| `no-ip-spoofing` | Blocks packets with a source IP not matching the VM's assigned IP |
| `no-arp-spoofing` | Prevents ARP replies with spoofed MAC/IP pairs |
| `allow-dhcp` | Permits DHCP client traffic |

### Applying a filter

```xml
<interface type='bridge'>
  <source bridge='virbr0'/>
  <filterref filter='clean-traffic'>
    <parameter name='IP' value='192.168.122.10'/>
  </filterref>
</interface>
```

### virsh nwfilter commands

```bash
virsh nwfilter-list                     # list all filters
virsh nwfilter-dumpxml clean-traffic    # show filter XML
virsh nwfilter-define myfilter.xml      # define / virsh nwfilter-undefine to remove
```

## Performance Features

### Multi-queue virtio-net

The [virtio-framework](virtio-framework.md) allows configuring multiple transmit/receive queue pairs on a single virtual NIC, enabling parallel packet processing across multiple vCPUs. This is critical for high-throughput workloads.

```xml
<interface type='bridge'>
  <source bridge='br0'/>
  <model type='virtio'/>
  <driver name='vhost' queues='4'/>
</interface>
```

The guest must also enable multi-queue on the interface:

```bash
ethtool -L eth0 combined 4
```

### vhost-net

The [vhost](vhost.md) kernel module (`vhost_net.ko`) moves virtio-net data plane processing from QEMU userspace into the host kernel. This eliminates context switches between QEMU and the kernel on every packet, significantly reducing latency and CPU overhead. vhost-net is enabled by default in modern libvirt/QEMU configurations via `<driver name='vhost'/>`.

### Jumbo frames

Configuring MTU 9000 on the bridge, physical NIC (`ip link set dev mtu 9000`), and inside the guest (`<mtu size='9000'/>` in the interface XML) reduces per-packet overhead for bulk transfers. All segments in the path -- physical NIC, bridge, and guest -- must agree on the MTU.

## See also

- [libvirt-management](libvirt-management.md)
- [virtio-framework](virtio-framework.md)
- [vhost](vhost.md)
- [bridging](bridging.md)
- [net-device](net-device.md)
- [qemu-kvm-overview](qemu-kvm-overview.md)
- [vm-live-migration](vm-live-migration.md)
- [vfio-device-passthrough](../analyses/analysis-vfio-device-passthrough.md)
