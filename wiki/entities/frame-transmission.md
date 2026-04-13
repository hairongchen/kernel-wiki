---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [understanding-linux-network-internals]
tags: [networking, frame-transmission, dev-queue-xmit, net-tx-action, queuing]
---

# Frame Transmission

The frame transmission subsystem handles sending packets from the protocol layer out through network devices. It manages queuing disciplines, device locking, and TX completion.

## Transmission Path

### dev_queue_xmit(skb)

The main entry point for frame transmission, called by the protocol layer (e.g., from `ip_finish_output2()` via the neighbor subsystem).

1. If the device has a queuing discipline (qdisc), enqueues the packet via `qdisc->enqueue()`
2. Calls `qdisc_run()` to attempt immediate dequeue and transmission
3. `qdisc_restart()` dequeues a packet and calls `dev->hard_start_xmit(skb, dev)` under the device's `xmit_lock`
4. If `hard_start_xmit()` returns failure (device busy), the packet is requeued

For devices with `tx_queue_len = 0` (virtual devices like loopback), the packet is sent directly without queuing discipline.

### hard_start_xmit(skb, dev)

The driver's transmit function. Transfers the packet to the hardware (typically via DMA to the NIC's TX ring buffer). Returns:
- `NETDEV_TX_OK` (0) — packet accepted
- `NETDEV_TX_BUSY` (1) — device busy, try again later

### Device Flow Control

Drivers use flow control functions to manage TX ring buffer fullness:

- `netif_stop_queue(dev)` — driver calls when TX ring is full; sets `__LINK_STATE_XOFF`, preventing further `hard_start_xmit()` calls
- `netif_wake_queue(dev)` — driver calls from TX completion interrupt when space becomes available; clears `__LINK_STATE_XOFF`, schedules `NET_TX_SOFTIRQ`
- `netif_queue_stopped(dev)` — tests `__LINK_STATE_XOFF`

## TX Completion (NET_TX_SOFTIRQ)

`net_tx_action()` is the `NET_TX_SOFTIRQ` handler. It performs two tasks:

1. **Free completed sk_buffs**: Drains the per-CPU `softnet_data.completion_queue`, freeing sk_buffs whose transmission is complete. Drivers add completed buffs here via `dev_kfree_skb_irq()` from their TX completion interrupt.

2. **Restart transmission**: Iterates the per-CPU `softnet_data.output_queue` (devices with pending TX work after `netif_wake_queue()`), calling `qdisc_run()` to resume dequeuing.

## Locking

- `xmit_lock` (per-device spinlock) — serializes access to `hard_start_xmit()`. Held during the actual device transmit call.
- `xmit_lock_owner` — CPU ID of the current lock holder; used to detect recursive calls (which would deadlock)
- On SMP, the lock ensures only one CPU transmits on a device at a time

## Watchdog Timer

Each device has a watchdog timer (`watchdog_timer`) set to fire after `watchdog_timeo` jiffies of TX inactivity. If the timer fires and the TX queue is stopped (but the device is up), the kernel calls `dev->tx_timeout()`, which typically resets the NIC hardware. This detects and recovers from stuck TX hardware.

## Enabling and Disabling Transmissions

- `dev_open()` enables transmission by activating the queuing discipline via `dev_activate()`
- `dev_close()` disables transmission via `dev_deactivate()`, which stops the qdisc and drains pending packets
- `netif_carrier_on()` / `netif_carrier_off()` — link detection; carrier loss stops transmission

## Key Functions Summary

| Function | Role |
|----------|------|
| `dev_queue_xmit(skb)` | Main TX entry point from protocol layer |
| `hard_start_xmit(skb, dev)` | Driver transmit function |
| `net_tx_action()` | NET_TX_SOFTIRQ handler; frees completed skbs, restarts queues |
| `netif_stop_queue(dev)` | Driver signals TX ring full |
| `netif_wake_queue(dev)` | Driver signals TX ring has space |
| `dev_kfree_skb_irq(skb)` | Queue skb for deferred freeing from IRQ context |

## See also

- [frame-reception](frame-reception.md)
- [sk-buff](sk-buff.md)
- [net-device](net-device.md)
- [concept-network-packet-flow](../concepts/concept-network-packet-flow.md)
