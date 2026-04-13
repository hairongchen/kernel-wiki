---
type: concept
created: 2026-04-09
updated: 2026-04-09
sources: [understanding-linux-network-internals]
tags: [networking, napi, interrupts, performance, livelock, polling]
---

# NAPI and Interrupt Mitigation

NAPI (New API) is a technique for high-performance network packet reception in the Linux kernel. It addresses the fundamental tension between low latency (interrupt-driven) and high throughput (polling) by dynamically switching between the two modes.

## The Problem: Interrupt Livelock

At high packet rates, pure interrupt-driven reception fails catastrophically:

1. Each arriving packet triggers a hardware interrupt
2. The CPU spends all its time servicing interrupts (top-half processing)
3. No CPU time remains for actual packet processing (bottom-half/application)
4. Packets pile up in the backlog queue and are eventually dropped
5. The system processes zero packets despite 100% CPU usage — **livelock**

This is a fundamental problem: the interrupt rate grows with packet rate, but processing capacity is fixed.

## The NAPI Solution

NAPI combines interrupts with polling:

1. **Low traffic**: Operate in interrupt mode — each packet triggers an interrupt for minimum latency
2. **High traffic**: Switch to polling mode — disable interrupts and poll the device in software, processing packets in batches
3. **Traffic subsides**: Switch back to interrupt mode

### State Machine

```
Interrupt Mode                     Polling Mode
    |                                   |
    | packet arrives                    | ring empty
    | interrupt fires                   | (all work done)
    v                                   v
[Disable IRQ] ----poll_list----> [net_rx_action polls]
    |                                   |
    | netif_rx_schedule()              | netif_rx_complete()
    |                                   | re-enable IRQ
    +-----------------------------------+
```

### Key Insight

In polling mode, packets accumulate in the NIC's hardware ring buffer. The poll function reads them in batches without interrupt overhead. The ring buffer acts as a natural accumulator — no kernel-side backlog queue is needed, and overflow management is handled by the NIC hardware (dropping at the source rather than wasting CPU cycles on packets that will be dropped later).

## Budget and Fairness

Two mechanisms prevent NAPI from monopolizing the CPU:

### Global Budget

`net_rx_action()` enforces:
- Maximum `netdev_budget` (default 300) packets per softirq invocation
- Maximum 2 jiffies of wall-clock time per invocation

If either limit is reached, the softirq re-raises itself and yields to other work.

### Per-Device Weight

Each NAPI device has a `weight` parameter (Ethernet default: 64). A single device can process at most `weight` packets per poll call. This prevents one high-traffic device from starving others on the same CPU.

## NAPI vs. Legacy (Non-NAPI)

| Aspect | NAPI | Legacy (netif_rx) |
|--------|------|-------------------|
| Interrupt handling | Disable after first, poll | Interrupt per packet |
| Packet storage | NIC ring buffer | Per-CPU backlog queue |
| Congestion behavior | Packets stay in NIC buffer | Packets dropped at queue limit |
| CPU overhead at high rates | Efficient (batch processing) | Livelock-prone |
| Latency at low rates | Same (interrupt-driven) | Same |
| Driver complexity | Higher (must implement poll) | Lower (just call netif_rx) |

## Comparison with Other Approaches

### Interrupt Coalescing (Hardware)

Some NICs support hardware interrupt coalescing — delivering one interrupt for multiple packets. This helps but:
- Requires hardware support
- Static configuration (can't adapt to traffic patterns dynamically)
- NAPI is complementary and works with any hardware

### Busy Polling

A newer technique (post-2.6) where the application polls the device directly in user context, bypassing softirq entirely. Lower latency than NAPI for latency-sensitive applications, but burns CPU.

## Implementation in the Networking Softirq

`net_rx_action()` is the `NET_RX_SOFTIRQ` handler:

```
void net_rx_action(struct softirq_action *h) {
    int budget = netdev_budget;
    unsigned long time_limit = jiffies + 2;

    while (!list_empty(&sd->poll_list)) {
        struct napi_struct *n = list_first_entry(...);
        int work = n->poll(n, min(n->weight, budget));
        budget -= work;
        if (budget <= 0 || time_after_eq(jiffies, time_limit))
            break;
    }
    /* If work remains, re-raise softirq */
    if (!list_empty(&sd->poll_list))
        __raise_softirq_irqoff(NET_RX_SOFTIRQ);
}
```

The per-CPU `softnet_data.poll_list` holds all NAPI-scheduled devices. The non-NAPI backlog is also on this list (via `backlog_dev`), unifying both paths.

## Relationship to Other Subsystems

- **Softirq infrastructure**: NAPI builds on `NET_RX_SOFTIRQ`. The softirq layer handles scheduling, CPU affinity, and the `ksoftirqd` fallback thread.
- **Protocol demultiplexing**: Both NAPI and legacy paths converge at `netif_receive_skb()` for protocol dispatch.
- **Traffic control**: Ingress qdisc hooks run within `netif_receive_skb()`, after NAPI but before protocol handlers.
- **Bridging**: Bridge hook (`br_handle_frame()`) runs inside `netif_receive_skb()` for bridged interfaces.

## See also

- [frame-reception](../entities/frame-reception.md)
- [interrupt-handling](../entities/interrupt-handling.md)
- [net-device](../entities/net-device.md)
- [sk-buff](../entities/sk-buff.md)
