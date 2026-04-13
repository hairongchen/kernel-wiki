---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [professional-linux-kernel-architecture]
tags: [networking, netfilter, iptables, firewall, connection-tracking, nat]
---

# Netfilter

Netfilter is the Linux kernel's packet filtering and manipulation framework. It provides a set of
hook points embedded in the networking stack where registered callback functions can inspect, modify,
drop, or queue packets as they traverse the IP layer. Rather than implementing filtering logic
directly in the core networking code, netfilter decouples policy from mechanism: the networking stack
simply invokes hooks at well-defined points, and loadable modules supply the actual filtering,
address translation, and connection tracking behavior.

The primary user-space interface for managing netfilter rules in the 2.6.24 kernel is **iptables**,
which communicates with the kernel via `setsockopt()`/`getsockopt()` calls on raw sockets to
install, list, and remove rule sets organized into tables and chains.

## Hook Points

Netfilter defines five hook points in the IPv4 packet processing path. Each hook corresponds to a
specific stage in the journey a packet takes through the kernel:

| Hook constant | Location in packet flow |
|---|---|
| `NF_INET_PRE_ROUTING` | After `ip_rcv()` completes basic sanity checks (checksum, version, header length) but **before** the routing decision |
| `NF_INET_LOCAL_IN` | After the routing decision determines the packet is destined for the **local host** |
| `NF_INET_FORWARD` | After the routing decision determines the packet must be **forwarded** to another host |
| `NF_INET_LOCAL_OUT` | For **locally generated** outbound packets, before routing |
| `NF_INET_POST_ROUTING` | The final hook before a packet **leaves the host**, applied to both forwarded packets and locally originated packets after routing |

These hooks divide the packet path into segments that correspond to distinct trust boundaries and
processing stages, allowing different policy modules (filtering, NAT, mangling) to act at the
appropriate point.

## Hook Registration: struct nf_hook_ops

Kernel modules register callbacks at specific hook points using `struct nf_hook_ops`:

- **`hook`** -- pointer to the callback function, with signature
  `unsigned int hook_fn(unsigned int hooknum, struct sk_buff *skb, const struct net_device *in, const struct net_device *out, int (*okfn)(struct sk_buff *))`
- **`pf`** -- protocol family (e.g., `PF_INET` for IPv4, `PF_INET6` for IPv6)
- **`hooknum`** -- which hook point this callback should be attached to
- **`priority`** -- integer controlling the order of callback invocation at a given hook point;
  lower values run first

Standard priority constants include:

| Priority | Value | Typical user |
|---|---|---|
| `NF_IP_PRI_FIRST` | INT_MIN | Conntrack helper, early processing |
| `NF_IP_PRI_CONNTRACK` | -200 | Connection tracking (inbound) |
| `NF_IP_PRI_MANGLE` | -150 | Mangle table |
| `NF_IP_PRI_NAT_DST` | -100 | Destination NAT (DNAT) |
| `NF_IP_PRI_FILTER` | 0 | Filter table |
| `NF_IP_PRI_NAT_SRC` | 100 | Source NAT (SNAT) |
| `NF_IP_PRI_CONNTRACK_CONFIRM` | INT_MAX | Conntrack confirmation (last) |

A module calls `nf_register_hook()` to add a single hook or `nf_register_hooks()` to add an array
of hooks atomically. Unregistration uses the corresponding `nf_unregister_hook()` /
`nf_unregister_hooks()` functions.

## The NF_HOOK() Macro

The core networking code invokes netfilter at each hook point through the `NF_HOOK()` macro (and its
variant `NF_HOOK_THRESH()`). Call sites include `ip_rcv()` (PRE_ROUTING), `ip_local_deliver()`
(LOCAL_IN), `ip_forward()` (FORWARD), `ip_output()` / `ip_queue_xmit()` (LOCAL_OUT), and
`ip_finish_output()` (POST_ROUTING).

`NF_HOOK()` iterates through all registered hooks at the specified hook point in priority order.
Each callback returns one of five verdicts:

| Verdict | Effect |
|---|---|
| `NF_ACCEPT` | Continue to the next hook, or proceed with normal processing if no hooks remain |
| `NF_DROP` | Discard the packet immediately; free the [sk-buff](sk-buff.md) |
| `NF_STOLEN` | The hook function has taken ownership of the packet; the caller must not touch the sk_buff again |
| `NF_QUEUE` | Pass the packet to user space via the `nf_queue` mechanism for asynchronous processing |
| `NF_REPEAT` | Call this same hook function again (used for iterative processing) |

If all hooks return `NF_ACCEPT`, the "okfn" (continuation function) is invoked to resume normal
packet processing.

## iptables

iptables organizes rules into **tables**, each containing **chains** that map to netfilter hook
points. The 2.6.24 kernel provides three default tables:

### filter Table

The standard packet filtering table. Contains three built-in chains:

- **INPUT** -- attached to `NF_INET_LOCAL_IN`; filters packets destined for local sockets
- **FORWARD** -- attached to `NF_INET_FORWARD`; filters packets being routed through the host
- **OUTPUT** -- attached to `NF_INET_LOCAL_OUT`; filters locally generated outbound packets

### nat Table

Performs Network Address Translation. Contains three built-in chains:

- **PREROUTING** -- attached to `NF_INET_PRE_ROUTING`; applies DNAT before routing
- **OUTPUT** -- attached to `NF_INET_LOCAL_OUT`; NAT for locally generated packets
- **POSTROUTING** -- attached to `NF_INET_POST_ROUTING`; applies SNAT/masquerading after routing

NAT rules are only evaluated for the first packet of a connection; subsequent packets in the same
tracked connection are translated automatically by the connection tracking subsystem.

### mangle Table

Modifies packet headers (TTL, TOS, marking). Has chains at all five hook points: PREROUTING, INPUT,
FORWARD, OUTPUT, and POSTROUTING.

### Internal Representation

Tables are represented in the kernel as `struct xt_table`. Each table contains an array of
`struct ipt_entry` elements, where each entry specifies:

- **Match criteria** -- standard fields (source/destination IP, interface, protocol) plus optional
  extended matches (`struct xt_match`) for port ranges, connection state, string patterns, etc.
- **Target** -- the action to take on a match (`struct xt_target`), such as ACCEPT, DROP, REJECT,
  SNAT, DNAT, MASQUERADE, LOG, or a jump to a user-defined chain

Rules are evaluated sequentially within a chain. The first matching rule's target determines the
packet's fate. If no rule matches, the chain's default policy applies.

## Connection Tracking (nf_conntrack)

The connection tracking subsystem (`nf_conntrack`) maintains state about active network connections,
enabling stateful filtering and NAT. It operates independently of iptables and can be used by
multiple netfilter modules.

### struct nf_conn

Each tracked connection is represented by a `struct nf_conn` with key fields:

- **`tuplehash[2]`** -- an array of two `struct nf_conntrack_tuple_hash` entries holding the
  original-direction and reply-direction tuples for the connection
- **`status`** -- bitmask of connection flags: `IPS_EXPECTED` (created by an expectation/helper),
  `IPS_CONFIRMED` (has seen packets in both directions or been confirmed at LOCAL_IN/POST_ROUTING),
  `IPS_SRC_NAT` / `IPS_DST_NAT` (NAT has been applied), `IPS_ASSURED` (enough traffic seen to
  survive early timeout), and others
- **`timeout`** -- timer for connection expiry; reset on each matching packet

### struct nf_conntrack_tuple

Identifies one direction of a connection:

- Source IP address and source-layer-4 identifier (port for TCP/UDP, id for ICMP)
- Destination IP address, destination-layer-4 identifier, and protocol number
- Direction (`IP_CT_DIR_ORIGINAL` or `IP_CT_DIR_REPLY`)

The connection tracking system maintains a **hash table** keyed by tuple values for efficient
lookup. When a packet arrives, its tuple is computed and looked up in the hash table to find or
create the corresponding `nf_conn`.

### Connection States

Connection tracking classifies packets into four states, accessible to iptables via the `state` or
`conntrack` match:

- **NEW** -- first packet of a connection (no existing conntrack entry)
- **ESTABLISHED** -- packet belongs to a connection that has seen traffic in both directions
- **RELATED** -- packet is starting a new connection that is associated with an existing one (e.g.,
  an FTP data connection related to an FTP control connection, identified by a conntrack helper)
- **INVALID** -- packet does not belong to any known connection and cannot be classified

### L4 Protocol Helpers

Connection tracking includes protocol-specific logic:

- **TCP** -- full state machine tracking (SYN, SYN-ACK, ACK, FIN, RST, etc.) with per-state
  timeouts and window tracking for validity checks
- **UDP** -- stateless protocol tracked via timeouts; a UDP "connection" is confirmed when a reply
  packet from the reverse direction is seen
- **ICMP** -- tracked by matching request/reply pairs using the ICMP id field

Application-layer helpers (`nf_conntrack_helper`) parse protocols like FTP, SIP, and H.323 to
identify dynamically negotiated secondary connections and create **expectations** for them.

### Hook Placement

Connection tracking registers at specific hook points:

- **PRE_ROUTING** and **LOCAL_OUT** (high priority) -- track new connections and look up existing
  ones as packets enter the netfilter pipeline
- **LOCAL_IN** and **POST_ROUTING** (low priority / `NF_IP_PRI_CONNTRACK_CONFIRM`) -- confirm
  connections as packets exit the pipeline, ensuring that only packets that survived all filtering
  are committed to the conntrack table

## NAT (Network Address Translation)

NAT in netfilter is built on top of connection tracking. When a NAT rule matches the first packet of
a connection, it records the translation in the `nf_conn` entry. All subsequent packets of that
connection are translated automatically without re-evaluating the NAT rules.

### SNAT and DNAT

- **DNAT (Destination NAT)** -- operates at the `NF_INET_PRE_ROUTING` hook (and `LOCAL_OUT` for
  locally generated packets). Modifies the destination address/port before routing, so the routing
  decision uses the translated address. Used for port forwarding and transparent proxying.
- **SNAT (Source NAT)** -- operates at the `NF_INET_POST_ROUTING` hook. Modifies the source
  address/port after routing. Used to allow multiple internal hosts to share a single external IP.

NAT translation state is stored in the conntrack entry (via `struct nf_nat_info`), including the
manipulated tuple for each direction.

### Masquerading

Masquerading is a variant of SNAT that automatically uses the primary IP address of the outgoing
network interface as the source address. This is useful for hosts with dynamically assigned IP
addresses (e.g., DHCP or PPP). If the interface's address changes, existing masqueraded connections
are invalidated.

## Packet Flow with Netfilter Hooks

The following diagram shows how netfilter hooks integrate with the IPv4 packet processing path:

```
Incoming packet
      |
      v
  ip_rcv()  -- sanity checks
      |
      v
 [NF_INET_PRE_ROUTING]  -- conntrack lookup, DNAT, mangle
      |
      v
  routing decision
     / \
    /   \
   v     v
 local?  forward?
   |       |
   v       v
[NF_INET_LOCAL_IN]    ip_forward()
   |                    |
   v                    v
 transport layer     [NF_INET_FORWARD]  -- filter (FORWARD chain)
 (TCP/UDP/ICMP)         |
   |                    v
   v               [NF_INET_POST_ROUTING]  -- SNAT, conntrack confirm
  local                 |
  processing            v
   |                  output device
   v
 locally generated response
   |
   v
[NF_INET_LOCAL_OUT]  -- conntrack, DNAT, filter (OUTPUT chain), mangle
   |
   v
 routing decision
   |
   v
[NF_INET_POST_ROUTING]  -- SNAT, conntrack confirm
   |
   v
 output device
```

This architecture ensures that DNAT happens before routing (so forwarding decisions use the
translated destination), SNAT happens after routing (so the correct outgoing interface is known),
and filtering happens at the appropriate trust boundaries.

## See also

- [ipv4-subsystem](ipv4-subsystem.md)
- [routing-subsystem](routing-subsystem.md)
- [sk-buff](sk-buff.md)
- [concept-network-packet-flow](../concepts/concept-network-packet-flow.md)
- [concept-linux-security](../concepts/concept-linux-security.md)
