---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [understanding-linux-network-internals]
tags: [networking, data-structure, net-device, network-interface, device-driver]
---

# Network Device (net_device)

`struct net_device` represents a network interface in the Linux kernel -- physical (Ethernet NIC, wireless adapter) or virtual (loopback, bridge, VLAN, tunnel). It is the central data structure connecting the device driver layer with the upper networking stack. Every interface visible via `ip link` or `ifconfig` corresponds to a `net_device` instance.

Defined in `include/linux/netdevice.h`.

## Key Fields

### Identification

| Field | Description |
|-------|-------------|
| `name[IFNAMSIZ]` | Interface name (e.g., "eth0", "br0", max 15 chars + null) |
| `ifindex` | Unique interface index (assigned at registration, never reused) |
| `type` | Hardware type (ARPHRD_ETHER, ARPHRD_LOOPBACK, etc.) |
| `flags` | Interface flags: IFF_UP, IFF_BROADCAST, IFF_LOOPBACK, IFF_PROMISC, IFF_MULTICAST, IFF_NOARP |

### Hardware Properties

| Field | Description |
|-------|-------------|
| `mtu` | Maximum transmission unit (default 1500 for Ethernet) |
| `hard_header_len` | L2 header length (14 for Ethernet) |
| `addr_len` | Hardware address length (6 for Ethernet) |
| `dev_addr[MAX_ADDR_LEN]` | Hardware (MAC) address |
| `broadcast[MAX_ADDR_LEN]` | Broadcast hardware address |
| `base_addr` | I/O base address |
| `irq` | Interrupt number |
| `mem_start` / `mem_end` | Shared memory range |

### State

| Field | Description |
|-------|-------------|
| `state` | Bit field: `__LINK_STATE_START` (dev_open called), `__LINK_STATE_PRESENT` (device exists), `__LINK_STATE_NOCARRIER` (no link), `__LINK_STATE_XOFF` (transmission stopped), `__LINK_STATE_SCHED` (on poll list) |
| `reg_state` | Registration state: NETREG_UNINITIALIZED, NETREG_REGISTERED, NETREG_UNREGISTERING, NETREG_RELEASED |
| `operstate` | RFC 2863 operational state: IF_OPER_UP, IF_OPER_DOWN, IF_OPER_DORMANT, etc. |
| `link_mode` | IF_LINK_MODE_DEFAULT or IF_LINK_MODE_DORMANT |

### Statistics

| Field | Description |
|-------|-------------|
| `get_stats()` | Function returning `struct net_device_stats` (rx/tx packets, bytes, errors, drops) |
| `last_rx` | Timestamp of last received packet |
| `trans_start` | Timestamp of last transmitted packet |

### Queuing

| Field | Description |
|-------|-------------|
| `qdisc` | Current queuing discipline (for traffic control) |
| `tx_queue_len` | Maximum transmit queue length |
| `watchdog_timer` | Transmission timeout watchdog |
| `watchdog_timeo` | Timeout value for watchdog |

### NAPI

| Field | Description |
|-------|-------------|
| `poll()` | NAPI poll function (driver-specific) |
| `poll_list` | Link in per-CPU NAPI poll list |
| `weight` | NAPI poll budget (typically 64 for Ethernet) |
| `quota` | Remaining budget for current poll cycle |

## Key Operations (Function Pointers)

| Function | Description |
|----------|-------------|
| `open()` | Called on `ifconfig up` / `ip link set up`. Allocates resources, enables interrupts. |
| `stop()` | Called on `ifconfig down`. Releases resources, disables interrupts. |
| `hard_start_xmit(skb, dev)` | Transmit a packet. Called by `dev_queue_xmit()`. Must be atomic. |
| `hard_header(skb, dev, type, daddr, saddr, len)` | Build the L2 header. For Ethernet: `eth_header()` |
| `rebuild_header(skb)` | Rebuild L2 header (used for ARP resolution retry) |
| `set_mac_address(dev, addr)` | Change hardware address |
| `do_ioctl(dev, ifr, cmd)` | Handle device-specific ioctl commands |
| `set_multicast_list(dev)` | Update multicast filter list |
| `change_mtu(dev, new_mtu)` | Change MTU |
| `tx_timeout(dev)` | Called when transmission times out |
| `get_stats(dev)` | Return device statistics |
| `neigh_setup(neigh_parms)` | Configure neighboring subsystem parameters for this device |
| `header_cache(neigh, hh)` | Fill L2 header cache entry |
| `header_cache_update(hh, dev, haddr)` | Update cached L2 header |

## Device Registration

### Registration Flow

1. **`alloc_netdev(sizeof_priv, name, setup_fn)`**: Allocate `net_device` + private driver data. `setup_fn` initializes type-specific defaults (e.g., `ether_setup()` for Ethernet).
2. **Driver fills in**: Function pointers, hardware properties, IRQ, memory.
3. **`register_netdevice(dev)`**:
   a. Assign `ifindex` (monotonically increasing)
   b. Add to global device list (`dev_base`) and name/index hash tables
   c. Set `reg_state = NETREG_REGISTERED`
   d. Send `NETDEV_REGISTER` notification on `netdev_chain`
4. **`NETDEV_UP` notification** sent when `dev_open()` is called.

### Unregistration Flow

1. **`unregister_netdevice(dev)`**:
   a. Send `NETDEV_UNREGISTER` notification
   b. Remove from hash tables
   c. Set `reg_state = NETREG_UNREGISTERING`
   d. Wait for all references to be released (`netdev_wait_allrefs()`)
   e. Free device structure

## Device State Transitions

```
alloc_netdev()       register_netdevice()      dev_open()
    |                       |                       |
    v                       v                       v
UNINITIALIZED ---------> REGISTERED -----------> IFF_UP set
                                                 __LINK_STATE_START set
                                                    |
                                                    | dev_close()
                                                    v
                                                 IFF_UP cleared
                                                    |
                            unregister_netdevice()  |
                                    |               |
                                    v               v
                            UNREGISTERING -------> freed
```

## Virtual Devices

Virtual `net_device` instances include:

| Type | Purpose | Creation |
|------|---------|----------|
| `lo` (loopback) | Local communication | Built-in, always present |
| `brN` (bridge) | L2 bridging | `brctl addbr` / `ip link add type bridge` |
| `vlanN` (802.1Q) | VLAN tagging | `vconfig` / `ip link add link ethX type vlan id N` |
| `tunN`/`tapN` | User-space tunnels | `ip tuntap add` |
| `gre`/`ipip` | IP tunnels | `ip tunnel add` |
| `bond0` | Link aggregation | `ip link add type bond` |

Virtual devices have `hard_start_xmit()` implementations that redirect packets through other real or virtual devices rather than to hardware.

## Interaction with Other Subsystems

- **Routing**: Routes reference `net_device` via `fib_nh.nh_dev`. Device up/down events trigger route table updates via `fib_netdev_event()`.
- **Neighboring**: Each device can have per-interface `neigh_parms`. `arp_constructor()` sets up neighbor operations based on device type.
- **Bridging**: Bridge slave devices have their `rx_handler` set to `br_handle_frame()`, intercepting packets before normal protocol processing.
- **Notification chains**: `netdev_chain` carries `NETDEV_UP`, `NETDEV_DOWN`, `NETDEV_REGISTER`, `NETDEV_UNREGISTER`, `NETDEV_CHANGEADDR`, etc.

## See also

- [sk-buff](sk-buff.md)
- [neighboring-subsystem](neighboring-subsystem.md)
- [routing-subsystem](routing-subsystem.md)
- [device-driver-model](device-driver-model.md)
- [concept-network-packet-flow](../concepts/concept-network-packet-flow.md)
- [src-understanding-linux-network-internals](../sources/src-understanding-linux-network-internals.md)
