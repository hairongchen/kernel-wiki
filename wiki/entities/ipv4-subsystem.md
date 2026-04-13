---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [understanding-linux-network-internals]
tags: [networking, ipv4, ip, routing, forwarding, fragmentation, icmp, l3]
---

# IPv4 Subsystem

The IPv4 subsystem implements the Internet Protocol version 4 in the Linux kernel. It handles packet reception, routing decisions, forwarding, local delivery, transmission, fragmentation/defragmentation, and ICMP.

## IP Header (struct iphdr)

| Field | Size | Description |
|-------|------|-------------|
| `version` | 4 bits | Always 4 |
| `ihl` | 4 bits | Header length in 32-bit words (min 5 = 20 bytes) |
| `tos` | 8 bits | Type of Service / DSCP + ECN |
| `tot_len` | 16 bits | Total packet length including header |
| `id` | 16 bits | Identification for fragment reassembly |
| `frag_off` | 16 bits | Flags (DF, MF) + fragment offset (in 8-byte units) |
| `ttl` | 8 bits | Time to Live, decremented per hop |
| `protocol` | 8 bits | Upper layer protocol (TCP=6, UDP=17, ICMP=1) |
| `check` | 16 bits | Header checksum (one's complement) |
| `saddr` | 32 bits | Source IP address |
| `daddr` | 32 bits | Destination IP address |

Options (0-40 bytes) follow if IHL > 5. Key options: Record Route, Timestamp, Loose/Strict Source Route, Router Alert.

## Reception Path

### ip_rcv() — Main Entry Point

Called from `netif_receive_skb()` via the `ip_packet_type` handler (ETH_P_IP). Performs initial validation:

- Verifies IP version is 4 and IHL >= 5
- Validates header checksum
- Ensures `tot_len` is consistent with skb length
- Trims padding added by L2 layer
- Passes to Netfilter `NF_IP_PRE_ROUTING` hook, then `ip_rcv_finish()`

### ip_rcv_finish() — Post-Netfilter Processing

- Calls `ip_route_input()` to perform routing lookup (sets `skb->dst`)
- Parses IP options via `ip_options_compile()` if IHL > 5
- Calls `dst_input(skb)` which invokes the function pointer set by routing:
  - `ip_local_deliver()` — destination is local
  - `ip_forward()` — destination is remote (forward)
  - `ip_mr_input()` — multicast routing

### Netfilter Hook Points

Five hooks are placed throughout the IPv4 path:

1. **NF_IP_PRE_ROUTING** — after basic validation, before routing decision
2. **NF_IP_LOCAL_IN** — before local delivery
3. **NF_IP_FORWARD** — before forwarding
4. **NF_IP_LOCAL_OUT** — after local packet generation
5. **NF_IP_POST_ROUTING** — before sending out on wire

## Forwarding

### ip_forward()

Handles packets routed through this host:

1. Checks TTL > 1 (sends ICMP Time Exceeded if TTL <= 1)
2. Checks packet size vs. outgoing MTU (if DF set and too large, sends ICMP Fragmentation Needed)
3. Decrements TTL
4. Incrementally updates IP header checksum
5. Passes through `NF_IP_FORWARD` hook
6. Calls `ip_forward_finish()` -> `ip_forward_options()` (updates Record Route, Timestamp, Source Route) -> `dst_output()` -> `ip_output()`

The kernel may send an ICMP Redirect if the packet goes out the same interface it came in on, suggesting the source use a different gateway.

## Local Delivery

### ip_local_deliver()

Handles packets destined for this host:

1. If packet is a fragment (MF set or fragment offset != 0), calls `ip_defrag()` for reassembly
2. Passes through `NF_IP_LOCAL_IN` hook
3. Calls `ip_local_deliver_finish()`

### ip_local_deliver_finish()

Dispatches to the upper-layer protocol handler:

1. Strips IP header (adjusts skb data pointer)
2. Delivers a copy to raw sockets via `raw_v4_input()`
3. Looks up `iphdr->protocol` in `inet_protos[]` array (O(1) by protocol number)
4. Calls the registered handler (`tcp_v4_rcv`, `udp_rcv`, `icmp_rcv`, etc.)
5. If no handler registered, sends ICMP Protocol Unreachable

### L4 Protocol Registration

Protocols register via `inet_add_protocol()` into the `inet_protos[256]` array (indexed by IP protocol number). Protected by RCU for lock-free read-side access. Each `net_protocol` entry provides:
- `handler` — receive function pointer
- `err_handler` — ICMP error handler
- `no_policy` — skip IPsec policy check

## Transmission

### Two Models

**TCP path** — `ip_queue_xmit()`:
- Called directly per segment
- TCP manages segmentation via MSS, so IP fragmentation is rare
- Performs route lookup, builds IP header, applies NF_IP_LOCAL_OUT

**UDP/raw path** — cork mechanism:
- `ip_append_data()` accumulates data into MTU-sized fragments in `sk->sk_write_queue`
- `ip_push_pending_frames()` builds IP headers and sends all fragments
- Handles fragmentation at accumulation time (more efficient than post-build fragmentation)
- `IP_CORK` socket option and `MSG_MORE` flag control corking behavior

### Output Path

Both models converge on `ip_output()`:

1. Sets output device on skb
2. Passes through `NF_IP_POST_ROUTING` hook
3. `ip_finish_output()` checks MTU; fragments if needed via `ip_fragment()`
4. `ip_finish_output2()` resolves L2 header:
   - If cached hardware header exists (`dst->hh`), uses fast `neigh_hh_output()`
   - Otherwise calls `dst->neighbour->output()` to trigger ARP if needed
5. Packet queued to device via `dev_queue_xmit()`

## Fragmentation and Defragmentation

### IP Fragmentation — ip_fragment()

Called when outgoing packet exceeds MTU (and DF bit is not set).

**Fast path**: If the skb already has a `frag_list` (pre-fragmented by `ip_append_data()`), adjusts IP headers on each fragment without copying data.

**Slow path**: Allocates new skbs, copies data in MTU-sized chunks. IP options with the "copied" flag are replicated to all fragments; others only in the first.

Each fragment gets: updated `tot_len`, correct `frag_off` (in 8-byte units), MF flag set on all but last.

### IP Defragmentation — ip_defrag()

Reassembles incoming fragments at the final destination.

**Fragment queue (`struct ipq`)**: Keyed by (src, dst, id, protocol) tuple. Hash table with periodic secret rekeying to prevent DoS. Each queue:
- `fragments` — ordered linked list of fragment skbs
- `len` — expected total length (known when last fragment arrives)
- `meat` — bytes received so far
- Expiration timer (default 30 seconds, `sysctl_ipfrag_time`)

**Memory management**: Global caps `sysctl_ipfrag_high_thresh` (default 256KB) and `sysctl_ipfrag_low_thresh` (default 192KB). `ip_evictor()` purges oldest queues under memory pressure.

**Overlap handling**: New fragments overlapping existing ones cause the older data to be trimmed (handles retransmissions and Teardrop-style attacks).

## ICMP (ICMPv4)

### Receiving — icmp_rcv()

Validates checksum, dispatches to type-specific handler via `icmp_control[type].handler()`:

| Type | Name | Handler |
|------|------|---------|
| 0 | Echo Reply | `icmp_discard()` |
| 3 | Destination Unreachable | `icmp_unreach()` |
| 5 | Redirect | `icmp_redirect()` |
| 8 | Echo Request | `icmp_echo()` |
| 11 | Time Exceeded | `icmp_unreach()` |

### Sending — icmp_send()

Called throughout the stack to generate ICMP error messages. Implements safety checks (RFC 1122):
- Never send ICMP about ICMP errors
- Never send ICMP about broadcast/multicast packets
- Never send ICMP about non-first fragments
- Never send ICMP about packets with 0.0.0.0 or broadcast source

Rate limiting via token bucket per destination (`xrlim_allow()`), controlled by `sysctl_icmp_ratelimit`.

### ICMP and Other Subsystems

- **Path MTU Discovery**: ICMP Fragmentation Needed (type 3, code 4) updates PMTU in routing cache via `ip_rt_frag_needed()`
- **Redirect**: ICMP Redirect (type 5) updates routing cache via `ip_rt_redirect()`
- **Error dispatch**: ICMP errors containing embedded packets are dispatched to L4 protocol `err_handler` via `inet_protos[]`

## IP Peer Information (inet_peer)

Per-remote-host cache stored in a global AVL tree:

- `v4daddr` — remote IPv4 address (lookup key)
- `ip_id_count` — per-destination IP ID counter (prevents ID prediction attacks)
- `tcp_ts` / `tcp_ts_stamp` — TCP timestamp for the peer

Created lazily via `inet_getpeer()`. Garbage collected when unused. `ip_select_ident()` uses the peer's counter for the IP ID field.

## IP-over-IP Tunneling

The `ipip` module implements protocol 4 (`IPPROTO_IPIP`). Creates virtual `tunl0` device. An outer IP header wraps the inner IP packet. `ipip_rcv()` strips the outer header and re-injects; `ipip_tunnel_xmit()` adds the outer header.

## Key /proc Tunables

| Path | Default | Description |
|------|---------|-------------|
| `/proc/sys/net/ipv4/ip_forward` | 0 | Enable/disable IP forwarding |
| `/proc/sys/net/ipv4/ip_default_ttl` | 64 | Default TTL for outgoing packets |
| `/proc/sys/net/ipv4/ipfrag_time` | 30 | Fragment reassembly timeout (seconds) |
| `/proc/sys/net/ipv4/ipfrag_high_thresh` | 262144 | Fragment memory high watermark |
| `/proc/sys/net/ipv4/ip_no_pmtu_disc` | 0 | Disable Path MTU Discovery |

## See also

- [routing-subsystem](routing-subsystem.md)
- [neighboring-subsystem](neighboring-subsystem.md)
- [arp](arp.md)
- [frame-reception](frame-reception.md)
- [frame-transmission](frame-transmission.md)
- [sk-buff](sk-buff.md)
- [concept-network-packet-flow](../concepts/concept-network-packet-flow.md)
