---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [professional-linux-kernel-architecture]
tags: [networking, tcp, udp, transport-layer, congestion-control, sockets]
---

# TCP and UDP Transport Layer Internals

## Overview

TCP (Transmission Control Protocol) and UDP (User Datagram Protocol) are the two primary transport protocols in the Linux networking stack. TCP provides reliable, ordered byte-stream delivery with flow control and congestion control. UDP provides unreliable datagram delivery with minimal overhead. Both are implemented as `struct proto` instances and dispatched from the IP layer via the `inet_protos[]` table based on protocol number.

In the Linux 2.6.24 kernel, the transport layer sits between the socket interface (BSD sockets / `struct sock`) above and the network layer ([ipv4-subsystem](ipv4-subsystem.md)) below. Outbound data flows from `sendmsg()` through the transport protocol into IP; inbound data arrives from IP's `ip_local_deliver()` and is dispatched to the appropriate transport handler.

## TCP Data Structures

### `struct tcp_sock`

`struct tcp_sock` extends `struct inet_connection_sock` (which extends `struct inet_sock`, which extends `struct sock`). It holds all TCP-specific per-connection state. Key fields include:

- **Congestion control**: `snd_cwnd` (congestion window in segments), `snd_ssthresh` (slow start threshold)
- **RTT estimation**: `srtt` (smoothed round-trip time), `mdev` (mean deviation), `mdev_max`, `rttvar` (RTT variance used for RTO calculation)
- **Sequence tracking**: `snd_una` (oldest unacknowledged sequence number), `snd_nxt` (next sequence number to send), `rcv_nxt` (next expected receive sequence number)
- **Flow control**: `rcv_wnd` (receive window advertised to peer), `snd_wnd` (send window from peer)
- **Buffers**: `out_of_order_queue` (segments received out of order), write queue for pending transmissions

### `struct tcp_skb_cb`

Each [sk-buff](sk-buff.md) carries a TCP control block in its `skb->cb[]` array, cast to `struct tcp_skb_cb`. This stores per-segment metadata:

- `seq` — starting sequence number of the segment
- `end_seq` — ending sequence number (seq + data length + SYN/FIN flags)
- `tcp_flags` — TCP header flags (SYN, ACK, FIN, RST, PSH, URG)
- `sacked` — SACK state flags for selective acknowledgment tracking
- `ack_seq` — acknowledgment number from the TCP header

### TCP State Machine

TCP connections follow the standard RFC 793 state machine. The current state is stored in `sk->sk_state` using constants such as `TCP_ESTABLISHED`, `TCP_SYN_SENT`, `TCP_CLOSE_WAIT`, and so on. The primary transitions are:

- **Active open**: CLOSED -> SYN_SENT -> ESTABLISHED
- **Passive open**: CLOSED -> LISTEN -> SYN_RECV -> ESTABLISHED
- **Active close**: ESTABLISHED -> FIN_WAIT_1 -> FIN_WAIT_2 -> TIME_WAIT -> CLOSED
- **Passive close**: ESTABLISHED -> CLOSE_WAIT -> LAST_ACK -> CLOSED

State transitions are driven by `tcp_rcv_state_process()` for non-ESTABLISHED states and `tcp_rcv_established()` for the common established-state fast path.

## TCP Connection Establishment

### Active Open

When a userspace process calls `connect()` on a TCP socket, the kernel invokes `tcp_v4_connect()`:

1. Route lookup to determine the outgoing interface and source address.
2. Allocate a local ephemeral port if one is not already bound.
3. Initialize sequence numbers and TCP options (window scale, SACK permitted, timestamps).
4. Build and transmit a SYN segment via `tcp_transmit_skb()`.
5. Transition the socket state to `TCP_SYN_SENT`.
6. Arm the retransmission timer in case the SYN is lost.

When the SYN-ACK arrives, `tcp_rcv_state_process()` handles it: the connection moves to `TCP_ESTABLISHED` and the waiting `connect()` call returns.

### Passive Open

On the listening side, inbound SYN segments follow this path:

1. `tcp_v4_rcv()` — main TCP receive entry point, looks up the socket by 4-tuple.
2. `tcp_v4_do_rcv()` — dispatches based on socket state.
3. `tcp_rcv_state_process()` — for a LISTEN socket, creates a `struct request_sock` to track the half-open connection and sends a SYN-ACK.
4. When the final ACK of the three-way handshake arrives, `tcp_v4_syn_recv_sock()` creates a full `struct sock` for the new connection and moves it to the accept queue.
5. The listening process retrieves it via `accept()`.

### SYN Cookie Protection

To defend against SYN flood attacks, the kernel supports SYN cookies (`tcp_syncookies` sysctl). When enabled and the SYN backlog overflows, the kernel encodes connection parameters into the SYN-ACK's initial sequence number rather than storing a `request_sock`. On receiving the client's ACK, the kernel reconstructs the connection state from the sequence number, avoiding resource exhaustion from half-open connections.

## TCP Transmission

### `tcp_sendmsg()`

When userspace calls `write()` or `send()` on a TCP socket, `tcp_sendmsg()` is invoked:

1. Copies user data into the socket's send buffer as a chain of `sk_buff` structures.
2. Coalesces data into an existing skb at the tail of the write queue when possible, avoiding allocation overhead.
3. Respects `sk->sk_sndbuf` to enforce send buffer limits; blocks or returns `EAGAIN` if the buffer is full.
4. Calls `tcp_push()` to trigger transmission when the Nagle algorithm permits.

### `tcp_write_xmit()`

This function drives actual segment transmission from the send queue:

1. Iterates over skbs in the write queue that have not yet been sent.
2. Checks the congestion window (`snd_cwnd`) and the receiver's advertised window (`snd_wnd`) to determine how many segments may be sent.
3. Performs TSO (TCP Segmentation Offload) segmentation if supported by the NIC.
4. Calls `tcp_transmit_skb()` for each segment.

### `tcp_transmit_skb()`

Builds the final TCP segment for transmission:

1. Constructs the TCP header (source/destination ports, sequence numbers, flags, window, urgent pointer).
2. Computes the TCP checksum (or sets up hardware checksum offload).
3. Adds TCP options (timestamps, SACK blocks, window scale).
4. Passes the skb to `ip_queue_xmit()` for IP-layer processing and routing.

### Nagle Algorithm

The Nagle algorithm (RFC 896) prevents sending many small segments by delaying transmission when there is unacknowledged data outstanding and the pending data is less than one MSS. This reduces overhead for interactive applications at the cost of minor latency. Applications that need low latency (e.g., real-time protocols) disable Nagle via the `TCP_NODELAY` socket option.

## TCP Reception

### Receive Path

Inbound TCP segments follow this path from the IP layer:

1. `tcp_v4_rcv()` — registered in `inet_protos[]` as the handler for protocol 6. Performs socket lookup using the 4-tuple (source IP, source port, destination IP, destination port).
2. `tcp_v4_do_rcv()` — dispatches to `tcp_rcv_established()` for connections in ESTABLISHED state (the fast path) or `tcp_rcv_state_process()` for other states.

### `tcp_rcv_established()` — Fast Path

For the common case of data exchange on established connections:

1. **Header prediction**: checks whether the segment is the next expected in-order segment with no special flags. If so, takes the fast path.
2. Processes ACK: advances `snd_una`, frees acknowledged skbs from the retransmit queue, and opens the congestion window.
3. Delivers in-order data directly to the socket's receive buffer (`sk->sk_receive_queue`).
4. Wakes up any process blocked in `recvmsg()`.

### Out-of-Order Handling

Segments that arrive out of order (sequence number does not match `rcv_nxt`) are placed in `tp->out_of_order_queue`, an RB-tree ordered by sequence number. When a missing segment arrives and fills a gap, contiguous segments from the out-of-order queue are merged into the receive queue. SACK information is generated to inform the sender which segments have been received, enabling targeted retransmission.

### Delayed ACKs

Rather than sending an ACK for every received segment, TCP uses delayed acknowledgments. An ACK is either piggy-backed on outbound data or sent after a short delay (typically 40ms). This reduces the number of pure ACK segments on the network. The delayed ACK timer is managed via `tcp_delack_timer()`. If two full-size segments arrive without an ACK being sent, an immediate ACK is triggered (RFC 2581 requirement).

## TCP Congestion Control

### Core Algorithms

TCP congestion control governs how fast a sender injects data into the network:

- **Slow start**: The congestion window (`snd_cwnd`) starts at 2-4 segments (Initial Window per RFC 3390). For each ACK received, cwnd increases by 1 MSS, resulting in exponential growth. Slow start continues until cwnd reaches `snd_ssthresh` or a loss is detected.

- **Congestion avoidance**: Once cwnd >= ssthresh, the sender enters congestion avoidance. The window increases by approximately 1 MSS per round-trip time (linear growth), implemented as `cwnd += 1/cwnd` for each ACK.

- **Fast retransmit**: When 3 duplicate ACKs are received (indicating a likely packet loss), the sender retransmits the missing segment immediately without waiting for the retransmission timer to expire.

- **Fast recovery** (RFC 3517): After fast retransmit, the sender sets `ssthresh = cwnd/2`, sets `cwnd = ssthresh + 3*MSS` (accounting for the 3 segments that triggered the duplicates), and continues in congestion avoidance mode rather than dropping back to slow start.

### Pluggable Congestion Framework

Linux 2.6.24 provides a pluggable congestion control framework via `struct tcp_congestion_ops`. Each algorithm registers callbacks:

- `ssthresh()` — compute new slow start threshold after loss
- `cong_avoid()` — adjust cwnd during normal transmission
- `cwnd_event()` — respond to specific events (e.g., entering/leaving fast recovery)
- `pkts_acked()` — called when packets are acknowledged, useful for delay-based algorithms

The default algorithm is selectable via sysctl (`net.ipv4.tcp_congestion_control`), and individual sockets can override it via the `TCP_CONGESTION` socket option. Common implementations include CUBIC (the default in many configurations), Reno, BIC, and Vegas.

## TCP Timers

TCP maintains several timers per connection:

- **Retransmission timer**: fires when an ACK for sent data is not received within the RTO (Retransmission Timeout). The RTO is computed from srtt and mdev and uses exponential backoff on successive timeouts (doubled each time, up to a maximum).
- **Delayed ACK timer**: fires after ~40ms to send a pending acknowledgment if no outbound data piggy-backs it.
- **Keepalive timer**: sends periodic probes on idle connections to detect dead peers (default 2 hours, configurable via `TCP_KEEPIDLE`).
- **TIME_WAIT timer**: holds the connection in TIME_WAIT state for 2 * MSL (Maximum Segment Lifetime, typically 60 seconds) to handle delayed segments.
- **Zero-window probe timer**: when the receiver advertises a zero window, the sender periodically sends 1-byte probes to detect when the window reopens.

## UDP Implementation

### `struct udp_sock`

`struct udp_sock` is a minimal extension of `struct inet_sock`. UDP maintains no connection state, no sequence numbers, and no congestion control — each datagram is independent.

### Transmission: `udp_sendmsg()`

When userspace sends a UDP datagram:

1. Resolves the destination address and performs a route lookup.
2. Builds an [sk-buff](sk-buff.md) containing the user payload.
3. Constructs the UDP header (source port, destination port, length, checksum).
4. Calls `ip_push_pending_frames()` or uses `ip_make_skb()` + `udp_send_skb()` to hand the datagram to the IP layer.
5. No buffering or retransmission — the datagram is sent immediately or an error is returned.

### Reception: `udp_rcv()`

Inbound UDP datagrams follow this path:

1. `udp_rcv()` is registered in `inet_protos[]` as the handler for protocol 17.
2. Calls `__udp4_lib_rcv()` which looks up the destination socket by destination port (and optionally source address for connected UDP sockets).
3. Validates the UDP checksum.
4. Delivers the datagram to the socket's receive queue via `sock_queue_rcv_skb()`.
5. If no matching socket is found, sends an ICMP "destination unreachable — port unreachable" message back to the sender.

### UDP-Lite

UDP-Lite (RFC 3828) is a variant that allows partial checksum coverage — only a specified prefix of the datagram is checksummed, permitting applications to receive partially corrupted data. Implemented via `struct udplite_sock` and registered as a separate protocol (IPPROTO_UDPLITE = 136).

## Protocol Registration

Both TCP and UDP are registered with the IP layer through `inet_protos[]`, an array indexed by protocol number:

- **TCP**: protocol number 6 (`IPPROTO_TCP`). The `struct net_protocol` entry points `tcp_v4_rcv()` as the receive handler. The `struct proto` instance `tcp_prot` provides socket-layer callbacks (`connect`, `sendmsg`, `recvmsg`, `close`, etc.).
- **UDP**: protocol number 17 (`IPPROTO_UDP`). The `struct net_protocol` entry points `udp_rcv()` as the receive handler. The `struct proto` instance `udp_prot` provides the socket-layer callbacks.

Registration happens during kernel initialization in `inet_init()`, which calls `inet_add_protocol()` for each transport protocol and `proto_register()` to register the `struct proto` instances with the socket layer.

## See also

- [socket-layer](socket-layer.md)
- [ipv4-subsystem](ipv4-subsystem.md)
- [sk-buff](sk-buff.md)
- [concept-network-packet-flow](../concepts/concept-network-packet-flow.md)
