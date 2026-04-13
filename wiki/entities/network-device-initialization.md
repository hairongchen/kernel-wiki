---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [understanding-linux-network-internals]
tags: [networking, device-initialization, pci, notification-chains, net-dev-init, registration]
---

# Network Device Initialization

This page covers the boot-time and runtime initialization of network devices in the Linux kernel, including notification chains, PCI bus interaction, device registration, and the `net_dev_init()` function.

## Notification Chains

The kernel uses a publish-subscribe mechanism for inter-subsystem event communication. A notification chain is a singly-linked list of `notifier_block` structures, each containing a callback function and a priority.

### notifier_block Structure

- `notifier_call` — callback function: `int (*)(struct notifier_block *, unsigned long event, void *data)`
- `next` — next block in the chain
- `priority` — execution order (higher = called first; default 0)

### Key Networking Chains

**`netdev_chain`** — network device events:

| Event | When |
|-------|------|
| `NETDEV_REGISTER` | Device registered with `register_netdevice()` |
| `NETDEV_UNREGISTER` | Device being unregistered |
| `NETDEV_UP` | Device opened (`dev_open()`) |
| `NETDEV_DOWN` | Device closed (`dev_close()`) |
| `NETDEV_CHANGE` | Device state changed |
| `NETDEV_CHANGEADDR` | Hardware address changed |
| `NETDEV_CHANGEMTU` | MTU changed |
| `NETDEV_CHANGENAME` | Device name changed |

**`inetaddr_chain`** — IPv4 address events:
- `NETDEV_UP` — IP address added to an interface
- `NETDEV_DOWN` — IP address removed from an interface

### Subscribers

- Routing subsystem (`fib_netdev_event`, `fib_inetaddr_event`) — creates/removes routes and flushes routing cache
- ARP (`arp_netdev_event`) — sends gratuitous ARP on address change
- Bridging — updates bridge port state
- Packet filter, bonding driver, IPsec

### API

- `notifier_chain_register(chain, block)` — subscribe
- `notifier_chain_unregister(chain, block)` — unsubscribe
- `notifier_call_chain(chain, event, data)` — invoke all callbacks; each returns `NOTIFY_OK`, `NOTIFY_DONE`, `NOTIFY_BAD`, or `NOTIFY_STOP`

## PCI Layer and NIC Drivers

### PCI Device Discovery

PCI provides automatic device discovery via bus enumeration. The kernel scans the PCI bus, reads vendor/device IDs from each device's configuration space, and matches them against registered `pci_driver` structures.

### pci_driver Structure

- `name` — driver name
- `id_table` — array of `pci_device_id` entries this driver handles (vendor, device, subvendor, subdevice, class)
- `probe(pdev, id)` — called when a matching device is found
- `remove(pdev)` — called when device is removed or driver unloaded
- `suspend` / `resume` — power management callbacks

### Typical NIC Driver Probe Sequence

1. `pci_enable_device()` — enable the PCI device, assign resources
2. `pci_request_regions()` — reserve I/O and memory regions
3. `pci_set_dma_mask()` — configure DMA addressing capability
4. `pci_set_master()` — enable PCI bus mastering (required for DMA)
5. `ioremap()` — map device registers into kernel virtual address space
6. `alloc_etherdev(sizeof_priv)` — allocate `net_device` + driver-private data
7. Populate `net_device` fields and function pointers
8. `register_netdev()` — register with the networking stack

### Power Management

NICs support power states D0 (fully on) through D3 (off). Wake-on-LAN (WOL) allows a NIC in low-power state to wake the system on receiving a magic packet.

## Device Registration

### net_dev_init()

Registered as `subsys_initcall` (runs before device drivers). Performs:

1. Initializes per-CPU `softnet_data` structures
2. Registers `NET_RX_SOFTIRQ` handler (`net_rx_action()`)
3. Registers `NET_TX_SOFTIRQ` handler (`net_tx_action()`)
4. Creates `/proc/net/dev` and related procfs entries

### register_netdev(dev) / register_netdevice(dev)

`register_netdev()` is the high-level wrapper that acquires the RTNL lock. `register_netdevice()` performs the actual work:

1. Calls `dev->init()` if defined
2. Assigns unique `ifindex` via `dev_new_index()` (monotonically increasing, never reused)
3. Resolves name patterns (e.g., "eth%d" -> "eth0") via `dev_alloc_name()`
4. Adds device to global hash tables (by name and by index)
5. Sets `__LINK_STATE_PRESENT` state bit
6. Sends `NETDEV_REGISTER` notification on `netdev_chain`

### alloc_etherdev(sizeof_priv)

Wrapper around `alloc_netdev()` that calls `ether_setup()` to set Ethernet defaults:
- `mtu` = 1500, `type` = `ARPHRD_ETHER`, `addr_len` = 6
- `tx_queue_len` = 1000
- `flags` = `IFF_BROADCAST | IFF_MULTICAST`
- `hard_header` = `eth_header`, `hard_header_cache` = `eth_header_cache`

### Device Open/Close

**`dev_open(dev)`:**
1. Calls driver's `open` function
2. Sets `IFF_UP` flag and `__LINK_STATE_START` bit
3. Activates queuing discipline via `dev_activate()`
4. Sends `NETDEV_UP` notification

**`dev_close(dev)`:**
1. Clears `__LINK_STATE_START`
2. Sends `NETDEV_GOING_DOWN` notification
3. Deactivates queuing discipline via `dev_deactivate()`
4. Calls driver's `stop` function
5. Clears `IFF_UP`
6. Sends `NETDEV_DOWN` notification

### Device State Flags

| Flag | Meaning |
|------|---------|
| `IFF_UP` | Administratively up (set by user) |
| `IFF_RUNNING` | Driver resources allocated, device operational |
| `__LINK_STATE_START` | Device has been opened (internal) |
| `IFF_PROMISC` | Promiscuous mode (reference counted via `promiscuity`) |
| `IFF_NOARP` | No ARP needed (loopback, tunnels) |

`IFF_UP && IFF_RUNNING` together = device fully operational.

### Device Lookup

- `dev_get_by_name(name)` — find by name, returns with incremented refcount
- `dev_get_by_index(ifindex)` — find by ifindex, returns with incremented refcount

## Kernel Init Infrastructure

### Initialization Call Levels

Functions marked with `__init` are placed in `.init.text` and freed after boot. The init levels execute in order:

1. `pure_initcall` — earliest
2. `core_initcall` — core kernel subsystems
3. `subsys_initcall` — subsystem init (e.g., `net_dev_init()`)
4. `device_initcall` — device drivers (default for `module_init` in built-in code)
5. `late_initcall` — runs last

This ordering ensures dependencies: networking core (`subsys_initcall`) initializes before NIC drivers (`device_initcall`).

### Module Init

- `module_init(fn)` — for modules: function called on `insmod`; for built-in: `device_initcall(fn)`
- `module_exit(fn)` — cleanup on `rmmod`; omitted for built-in code

## See also

- [net-device](net-device.md)
- [frame-reception](frame-reception.md)
- [device-driver-model](device-driver-model.md)
- [interrupt-handling](interrupt-handling.md)
- [kernel-modules](kernel-modules.md)
