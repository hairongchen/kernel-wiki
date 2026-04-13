---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [understanding-linux-network-internals]
tags: [networking, data-structure, sk-buff, packet, socket-buffer]
---

# Socket Buffer (sk_buff)

`struct sk_buff` is the fundamental data structure representing a network packet in the Linux kernel. Every packet -- received from the network, generated locally, or being forwarded -- is wrapped in an `sk_buff`. It carries not just the packet data but also metadata needed by every layer of the networking stack: protocol headers, routing decisions, device references, timestamps, and control flags.

Defined in `include/linux/skbuff.h`, implemented in `net/core/skbuff.c`.

## Structure Overview

An `sk_buff` consists of two parts:
1. **The `sk_buff` structure itself**: Metadata, pointers, flags (~200 bytes)
2. **The data buffer**: The actual packet bytes (allocated separately)

```
sk_buff structure              Data buffer
+------------------+          +------------------+
| next, prev       |          | headroom         |  <-- head
| sk (socket)      |          |                  |
| dev (device)     |          +------------------+
| dst (routing)    |          | data region      |  <-- data
| ...              |    ----->|  [MAC header]    |  <-- mac.raw
| head  -----------+----|     |  [IP header]     |  <-- nh.raw
| data  -----------+-------->|  [TCP/UDP hdr]   |  <-- h.raw
| tail  -----------+-------->|  [payload]       |
| end   -----------+----     |                  |  <-- tail
| len              |    ---->+------------------+
| data_len         |          | tailroom         |  <-- end
| ...              |          +------------------+
+------------------+
```

## Key Fields

### Buffer Management

| Field | Description |
|-------|-------------|
| `head` | Start of allocated buffer |
| `data` | Start of current data (moves as headers are pushed/pulled) |
| `tail` | End of current data |
| `end` | End of allocated buffer |
| `len` | Total data length (linear + paged fragments) |
| `data_len` | Length of paged (non-linear) data |
| `truesize` | Total memory consumed (sk_buff + buffer + fragments) |

### Protocol Headers

| Field | Description |
|-------|-------------|
| `mac.raw` | Pointer to L2 (MAC/Ethernet) header |
| `nh.raw` / `nh.iph` | Pointer to L3 (network/IP) header |
| `h.raw` / `h.th` / `h.uh` | Pointer to L4 (transport/TCP/UDP) header |

### Packet Metadata

| Field | Description |
|-------|-------------|
| `dev` | Device the packet was received on (ingress) or will be sent through (egress) |
| `input_dev` | Device packet was originally received on (preserved across forwarding) |
| `dst` | Destination cache entry (`dst_entry` from routing lookup) |
| `sk` | Owning socket (NULL for forwarded/locally-generated-without-socket packets) |
| `protocol` | L3 protocol (ETH_P_IP, ETH_P_ARP, etc.), set by driver |
| `pkt_type` | Packet type: PACKET_HOST, PACKET_BROADCAST, PACKET_MULTICAST, PACKET_OTHERHOST |
| `ip_summed` | Checksum status: CHECKSUM_NONE, CHECKSUM_HW, CHECKSUM_UNNECESSARY |
| `priority` | QoS priority (from TOS field or socket option) |
| `nfmark` | Netfilter mark (for iptables MARK target) |
| `stamp` | Receive timestamp |

### List Management

| Field | Description |
|-------|-------------|
| `next` / `prev` | Doubly-linked list pointers (for `sk_buff_head` queues) |
| `list` | Pointer to owning `sk_buff_head` |

### Reference Counting

| Field | Description |
|-------|-------------|
| `users` | Reference count (atomic). When it reaches zero, the sk_buff is freed |
| `cloned` | This sk_buff was cloned (shares data buffer with another sk_buff) |
| `dataref` | Reference count on the data buffer (in `skb_shared_info`) |

## Key Operations

### Buffer Manipulation

| Function | Description |
|----------|-------------|
| `alloc_skb(size, gfp)` | Allocate new sk_buff + data buffer |
| `dev_alloc_skb(size)` | Allocate for device driver use (GFP_ATOMIC, includes headroom) |
| `kfree_skb(skb)` | Decrement refcount, free if zero |
| `skb_reserve(skb, len)` | Increase headroom (move data and tail forward). Must be called before any data is added. |
| `skb_put(skb, len)` | Add data to tail, return pointer to start of new area |
| `skb_push(skb, len)` | Add data to head (prepend header), return new data pointer |
| `skb_pull(skb, len)` | Remove data from head (strip header), return new data pointer |
| `skb_headroom(skb)` | Available headroom (data - head) |
| `skb_tailroom(skb)` | Available tailroom (end - tail) |

### Cloning and Copying

| Function | Description |
|----------|-------------|
| `skb_clone(skb, gfp)` | Create a copy of the sk_buff structure sharing the same data buffer. Used when multiple consumers need the same packet (e.g., protocol sniffers + normal processing) |
| `skb_copy(skb, gfp)` | Create a full copy (new sk_buff + new data buffer) |
| `pskb_copy(skb, gfp)` | Copy sk_buff + linear data, share paged fragments |
| `skb_cow(skb, headroom)` | Copy-on-write: if cloned or insufficient headroom, make a private copy |

### Queue Operations

Packets are often managed in doubly-linked queues (`struct sk_buff_head`):

| Function | Description |
|----------|-------------|
| `skb_queue_head(list, skb)` | Add to front of queue |
| `skb_queue_tail(list, skb)` | Add to end of queue |
| `skb_dequeue(list)` | Remove from front of queue |
| `skb_queue_purge(list)` | Free all sk_buffs in queue |
| `skb_queue_len(list)` | Number of entries |

## Lifecycle Example (Ingress)

1. **Driver receives frame**: Allocates `sk_buff` via `dev_alloc_skb()`, copies frame data from NIC ring buffer, sets `skb->protocol`, `skb->dev`, `skb->pkt_type`.
2. **Protocol demux**: `netif_receive_skb()` dispatches based on `skb->protocol` to `ip_rcv()`.
3. **IP processing**: `ip_rcv()` sets `skb->nh.iph`, validates header. `ip_rcv_finish()` performs routing lookup, sets `skb->dst`.
4. **Routing decision**: `skb->dst->input()` called -- either `ip_local_deliver()` or `ip_forward()`.
5. **Transport delivery**: `ip_local_deliver_finish()` strips IP header via `skb_pull()`, dispatches to TCP/UDP handler.
6. **Socket queue**: Transport layer queues `sk_buff` on `sk->sk_receive_queue`.
7. **Application read**: `recv()`/`read()` copies data to user space, `kfree_skb()` frees the buffer.

## Memory Layout Optimization

- **`NET_SKB_PAD`**: 16 bytes of padding reserved at the start of the data buffer for possible header prepending (e.g., adding an Ethernet header for bridging).
- **`SKB_DATA_ALIGN`**: Data buffer size is cache-line aligned.
- **`skb_shared_info`**: Placed at `end` of the data buffer. Contains fragment array (`skb_frag_t[]`) for scatter-gather I/O, `frag_list` for chained buffers, `dataref` for shared data reference counting.
- **Paged data**: For large packets (TSO, jumbo frames), data can span multiple pages via `skb_shared_info.frags[]` rather than requiring a single contiguous buffer.

## See also

- [net-device](net-device.md)
- [routing-subsystem](routing-subsystem.md)
- [concept-network-packet-flow](../concepts/concept-network-packet-flow.md)
- [src-understanding-linux-network-internals](../sources/src-understanding-linux-network-internals.md)
