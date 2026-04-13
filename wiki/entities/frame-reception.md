---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [understanding-linux-network-internals]
tags: [networking, frame-reception, napi, interrupts, softirq, net-rx-action]
---

# Frame Reception

The frame reception subsystem handles the path from NIC hardware interrupt through protocol demultiplexing. It is one of the most performance-critical paths in the kernel, and the design evolved significantly with the introduction of NAPI to prevent interrupt livelock under high packet rates.

## Key Data Structures

### softnet_data (per-CPU)

Each CPU has its own `softnet_data` instance, eliminating cross-CPU contention on the receive path.

- `poll_list` — list of NAPI-scheduled devices with pending RX work
- `input_pkt_queue` — backlog queue for non-NAPI drivers (packets enqueued by `netif_rx()`)
- `completion_queue` — sk_buffs waiting to be freed (deferred for performance)
- `output_queue` — devices with pending TX completion work
- `backlog_dev` — virtual NAPI device whose `poll` function (`process_backlog()`) drains `input_pkt_queue`

Initialized by `net_dev_init()`, which is a `subsys_initcall`.

### packet_type

Protocol handlers registered via `dev_add_pack()`. Each entry specifies:

- `type` — protocol ID (e.g., `ETH_P_IP` = 0x0800, `ETH_P_ARP` = 0x0806)
- `dev` — specific device or NULL for all devices
- `func` — handler function (e.g., `ip_rcv()` for IPv4, `arp_rcv()` for ARP)

Stored in a hash table (`ptype_base[]`) and a catch-all list (`ptype_all` for packet sniffers).

## NAPI (New API) Reception

NAPI was introduced to solve interrupt livelock — at very high packet rates, the CPU can spend all its time handling hardware interrupts and never process packets in the backlog.

### Mechanism

1. **First packet arrives**: NIC raises hardware interrupt
2. **Top-half handler**: Acknowledges interrupt, disables further RX interrupts from device, calls `netif_rx_schedule(dev)` to add device to the CPU's `poll_list` and raise `NET_RX_SOFTIRQ`
3. **Softirq fires**: `net_rx_action()` iterates the `poll_list`, calling each device's `poll()` method
4. **poll() method**: Reads packets from device ring buffer, builds sk_buffs, calls `netif_receive_skb()` for each. Returns number of packets processed.
5. **Completion**: When the device ring is empty, `poll()` calls `netif_rx_complete(dev)` to remove from `poll_list` and re-enables hardware RX interrupts

### Budget Mechanism

`net_rx_action()` enforces two limits to prevent CPU starvation:

- **Packet budget**: `netdev_budget` (default 300) total packets across all devices per softirq invocation
- **Time limit**: 2 jiffies maximum per invocation
- **Per-device weight**: Each NAPI device has a `weight` parameter (Ethernet typically 64) controlling how many packets it processes per poll call

If the budget is exhausted before all devices are drained, `net_rx_action()` raises `NET_RX_SOFTIRQ` again to continue processing.

### NAPI Driver Pattern

**Interrupt handler:**
```
irqreturn_t my_interrupt(int irq, void *dev_id) {
    struct net_device *dev = dev_id;
    /* Acknowledge hardware interrupt */
    disable_rx_interrupts(dev);
    netif_rx_schedule(dev);
    return IRQ_HANDLED;
}
```

**Poll function:**
```
int my_poll(struct net_device *dev, int *budget) {
    int work = 0, quota = min(dev->quota, *budget);
    while (work < quota && packets_pending(dev)) {
        skb = read_packet_from_ring(dev);
        netif_receive_skb(skb);
        work++;
    }
    *budget -= work;
    dev->quota -= work;
    if (!packets_pending(dev)) {
        netif_rx_complete(dev);
        enable_rx_interrupts(dev);
        return 0;
    }
    return 1;  /* more work to do */
}
```

## Legacy (Non-NAPI) Reception

Older drivers that don't implement NAPI use the `netif_rx()` path:

1. **IRQ handler** builds an sk_buff and calls `netif_rx(skb)`
2. `netif_rx()` enqueues the skb on the CPU's `softnet_data.input_pkt_queue`
3. If the queue wasn't already scheduled, adds `backlog_dev` to the `poll_list` and raises `NET_RX_SOFTIRQ`
4. `net_rx_action()` calls `process_backlog()` (the `poll` function for `backlog_dev`)
5. `process_backlog()` dequeues packets and calls `netif_receive_skb()` for each

`netif_rx()` returns congestion status: `NET_RX_SUCCESS`, `NET_RX_CN_LOW/MOD/HIGH`, or `NET_RX_DROP` (queue full, packet dropped).

## Protocol Demultiplexing

`netif_receive_skb()` is the central dispatch function, called for every received frame regardless of NAPI vs. legacy path:

1. Sets `skb->tstamp` (packet timestamp)
2. Delivers a copy to all `ptype_all` handlers (packet sniffers like tcpdump)
3. If the device is part of a bridge, calls `br_handle_frame()` for bridging decision
4. Looks up `skb->protocol` in `ptype_base[]` hash table
5. Calls the matching protocol handler's `func()` (e.g., `ip_rcv()` for IPv4, `arp_rcv()` for ARP)

### Protocol Handler Registration

- `dev_add_pack(pt)` — registers a `packet_type` handler
- `dev_remove_pack(pt)` — unregisters a handler
- Handlers registered at boot time (e.g., `ip_init()` registers `ip_packet_type` for ETH_P_IP)

### Ethernet vs. IEEE 802.3

`netif_receive_skb()` handles both frame types. Ethernet II frames have a 2-byte protocol field (values >= 0x0600). IEEE 802.3 frames have a 2-byte length field (values < 0x0600) followed by LLC/SNAP headers. The kernel determines the type by checking whether `skb->protocol` is >= `ETH_P_802_3_MIN` (0x0600).

## Congestion Management

The non-NAPI path implements per-CPU congestion tracking:

- Each CPU's `input_pkt_queue` has a length limit (`netdev_max_backlog`, default 1000)
- When the queue exceeds the limit, packets are dropped at `netif_rx()`
- Congestion levels (LOW/MOD/HIGH) are reported to drivers, which may reduce their processing rate
- NAPI inherently handles congestion by leaving packets in the device ring buffer until the poll function processes them

## Key Functions Summary

| Function | Role |
|----------|------|
| `net_rx_action()` | NET_RX_SOFTIRQ handler; polls NAPI devices |
| `netif_rx_schedule(dev)` | Schedule NAPI device for polling |
| `netif_rx_complete(dev)` | Remove device from poll list |
| `netif_rx(skb)` | Legacy non-NAPI frame reception |
| `netif_receive_skb(skb)` | Protocol demultiplexing dispatch |
| `process_backlog()` | Poll function for non-NAPI backlog |
| `dev_add_pack(pt)` | Register protocol handler |

## See also

- [sk-buff](sk-buff.md)
- [net-device](net-device.md)
- [frame-transmission](frame-transmission.md)
- [interrupt-handling](interrupt-handling.md)
- [ipv4-subsystem](ipv4-subsystem.md)
- [concept-napi-interrupt-mitigation](../concepts/concept-napi-interrupt-mitigation.md)
