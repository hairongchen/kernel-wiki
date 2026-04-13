---
type: concept
created: 2026-04-09
updated: 2026-04-09
sources: [understanding-linux-network-internals]
tags: [networking, packet-flow, ipv4, routing, bridging]
---

# Network Packet Flow

This page describes how packets flow through the Linux kernel's networking stack, from NIC hardware interrupt to application delivery (ingress) or from socket to wire (egress). Understanding this flow is essential for grasping how the various networking subsystems -- device drivers, protocol handlers, bridging, IP, routing, neighboring, and transport -- interact.

## Ingress Path (Packet Reception)

```
NIC hardware interrupt
        |
        v
  Driver interrupt handler
  (allocates sk_buff, copies data, records protocol)
        |
        v
  netif_rx() or napi_schedule()
  (queue to per-CPU backlog or schedule NAPI poll)
        |
        v
  NET_RX_SOFTIRQ: net_rx_action()
        |
        v
  netif_receive_skb()
  (protocol demux based on skb->protocol)
        |
        +-------> Bridge hook: br_handle_frame()  [if bridged interface]
        |              |
        |              +-> forward / flood / deliver locally
        |
        +-------> Protocol handler dispatch
                  (dev_add_pack registered handlers)
                       |
                       v
                  ip_rcv()  [ETH_P_IP]
                       |
                       v
                  NF_IP_PRE_ROUTING hook
                       |
                       v
                  ip_rcv_finish()
                       |
                       v
                  ip_route_input()  --> routing cache lookup
                       |                    |
                       |               cache miss: ip_route_input_slow()
                       |                    |
                       |               fib_lookup() --> FIB
                       |                    |
                       |               create rtable, rt_intern_hash()
                       |
                       v
                  skb->dst->input()
                       |
            +----------+----------+
            |                     |
     ip_local_deliver()     ip_forward()
            |                     |
            v                     v
     NF_IP_LOCAL_IN        NF_IP_FORWARD
            |                     |
            v                     v
     ip_local_deliver_      ip_forward_finish()
     finish()                     |
            |                     v
            v               dst_output()
     Transport layer              |
     (TCP/UDP/ICMP               v
      via inet_protos[])    ip_output()
            |                     |
            v                     v
     sk_receive_queue       [continues to egress path]
            |
            v
     Application recv()
```

## Egress Path (Packet Transmission)

```
Application send() / kernel generates packet
        |
        v
  Transport layer builds segment
  (TCP: tcp_transmit_skb, UDP: udp_sendmsg)
        |
        v
  ip_queue_xmit() or ip_push_pending_frames()
        |
        v
  ip_route_output_key()  --> routing cache lookup
        |                         |
        |                    cache miss: ip_route_output_slow()
        |                         |
        |                    source addr selection + fib_lookup()
        |                         |
        |                    create rtable, rt_intern_hash()
        |
        v
  Build IP header (ip_build_and_send_pkt or ip_queue_xmit internals)
        |
        v
  NF_IP_LOCAL_OUT hook
        |
        v
  dst_output() --> skb->dst->output()
        |
        v
  ip_output()
        |
        v
  NF_IP_POST_ROUTING hook
        |
        v
  ip_finish_output()
        |
        +-> if packet > MTU: ip_fragment()
        |
        v
  ip_finish_output2()
        |
        v
  dst->neighbour->output()
  (resolves L2 address via ARP if needed)
        |
        v
  neigh_resolve_output() or neigh_connected_output()
        |
        v
  dev_queue_xmit()
        |
        v
  Queuing discipline (if configured)
        |
        v
  dev->hard_start_xmit()
        |
        v
  NIC driver transmits frame
```

## Key Decision Points

### 1. Protocol Demultiplexing

`netif_receive_skb()` examines `skb->protocol` (set by the driver from the Ethernet type field) and dispatches to the registered protocol handler:

| Protocol | Handler | Constant |
|----------|---------|----------|
| IPv4 | `ip_rcv()` | `ETH_P_IP` (0x0800) |
| ARP | `arp_rcv()` | `ETH_P_ARP` (0x0806) |
| IPv6 | `ipv6_rcv()` | `ETH_P_IPV6` (0x86DD) |
| 802.1Q VLAN | `vlan_skb_recv()` | `ETH_P_8021Q` (0x8100) |

### 2. Routing Decision

`ip_route_input()` determines the packet's fate by setting `skb->dst->input`:

| Route Type | `input` function | Behavior |
|------------|------------------|----------|
| `RTN_LOCAL` | `ip_local_deliver` | Deliver to local transport layer |
| `RTN_UNICAST` | `ip_forward` | Forward to another host |
| `RTN_BROADCAST` | `ip_local_deliver` | Deliver locally (broadcast) |
| `RTN_MULTICAST` | `ip_mr_input` | Multicast routing |
| `RTN_UNREACHABLE` | `ip_error` | Drop + ICMP unreachable |
| `RTN_BLACKHOLE` | `dst_discard` | Silently drop |

### 3. L3-to-L2 Handoff

The transition from network layer to link layer happens through the [neighboring-subsystem](../entities/neighboring-subsystem.md):

- The `dst_entry` holds a `neighbour` pointer (bound during `rt_intern_hash()` via `arp_bind_neighbour()`)
- `ip_finish_output2()` calls `dst->neighbour->output()`
- If the neighbor is in `NUD_CONNECTED` state: `neigh_connected_output()` -- fast path, L2 header is known
- If not connected: `neigh_resolve_output()` -- may need to send ARP request, packet queued in `arp_queue`
- With L2 header caching (`hh_cache`): `neigh_hh_output()` -- fastest path, copies pre-built header

### 4. Forwarding Path

When forwarding (`ip_forward()`):
1. Decrement TTL; if zero, send ICMP Time Exceeded and drop
2. Check if packet exceeds next-hop MTU; if so, and DF bit is set, send ICMP Fragmentation Needed and drop
3. Apply `NF_IP_FORWARD` netfilter hook
4. Check if ICMP redirect should be sent (packet leaving same interface it arrived on)
5. Call `ip_forward_finish()` -> `dst_output()` -> `ip_output()`

## Netfilter Hook Points

Five hook points exist in the IPv4 path, each allowing registered hooks (iptables rules) to inspect/modify/drop packets:

| Hook | Location | Chains |
|------|----------|--------|
| `NF_IP_PRE_ROUTING` | After `ip_rcv()`, before routing decision | PREROUTING |
| `NF_IP_LOCAL_IN` | After routing, for locally-destined packets | INPUT |
| `NF_IP_FORWARD` | For forwarded packets | FORWARD |
| `NF_IP_LOCAL_OUT` | For locally-generated packets, after routing | OUTPUT |
| `NF_IP_POST_ROUTING` | Final hook before output to device | POSTROUTING |

## NAPI and Interrupt Mitigation

Modern drivers use NAPI to handle high packet rates:

1. First packet triggers hardware interrupt
2. Driver's interrupt handler calls `napi_schedule()` -- disables further interrupts, adds device to poll list
3. `net_rx_action()` softirq calls device's `poll()` method
4. `poll()` processes packets from ring buffer, calling `netif_receive_skb()` for each
5. When ring buffer is empty or budget exhausted, re-enables interrupts via `napi_complete()`

This **interrupt coalescing** prevents livelock under high load by switching from interrupt-driven to polling mode.

## Per-CPU Processing

The networking stack uses per-CPU data to minimize locking:

- **`softnet_data`**: Per-CPU structure with input queue, NAPI poll list, completion queue, backlog device
- Packets are typically processed entirely on the CPU where the interrupt occurred
- `netif_rx()` enqueues to the local CPU's backlog
- `net_rx_action()` processes the local CPU's poll list

## See also

- [routing-subsystem](../entities/routing-subsystem.md)
- [routing-cache](../entities/routing-cache.md)
- [neighboring-subsystem](../entities/neighboring-subsystem.md)
- [arp](../entities/arp.md)
- [sk-buff](../entities/sk-buff.md)
- [net-device](../entities/net-device.md)
- [src-understanding-linux-network-internals](../sources/src-understanding-linux-network-internals.md)
