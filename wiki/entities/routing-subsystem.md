---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [understanding-linux-network-internals]
tags: [networking, routing, ipv4, fib, forwarding]
---

# Routing Subsystem

The Linux routing subsystem determines how IP packets are forwarded -- which interface they leave from, what next-hop gateway to use, and whether they should be delivered locally, forwarded, or dropped. It is organized as a two-level architecture: a fast-path **routing cache** and a slow-path **Forwarding Information Base (FIB)**. The routing subsystem lives primarily in `net/ipv4/route.c` (cache), `net/ipv4/fib_frontend.c` (FIB front-end), and backend-specific files.

## Architecture Overview

```
Packet arrives / Socket wants to send
            |
            v
   +------------------+
   |  Routing Cache    |  <-- Fast path: hash table lookup
   |  (struct rtable)  |      O(1) average
   +------------------+
      |  hit     |  miss
      v          v
   use cached   +------------------+
   result       |  FIB Lookup       |  <-- Slow path: longest prefix match
                |  (fib_lookup())   |
                +------------------+
                         |
                         v
                  Create rtable entry
                  Insert into cache
                         |
                         v
                  Return result
```

### Lookup Dimensions

A routing decision considers:
- **Destination address** (always)
- **Source address** (for policy routing and reverse path filtering)
- **TOS** (Type of Service field)
- **Input interface** (for ingress routing)
- **Output interface** (when specified by socket)
- **Fwmark** (firewall mark, for policy routing)

### Route Types

The result of a routing lookup classifies the packet:

| Type | Constant | Meaning |
|------|----------|---------|
| Unicast | `RTN_UNICAST` | Forward to next hop |
| Local | `RTN_LOCAL` | Deliver locally |
| Broadcast | `RTN_BROADCAST` | Deliver locally + broadcast |
| Multicast | `RTN_MULTICAST` | Multicast delivery |
| Unreachable | `RTN_UNREACHABLE` | Drop, send ICMP unreachable |
| Blackhole | `RTN_BLACKHOLE` | Silently drop |
| Prohibit | `RTN_PROHIBIT` | Drop, send ICMP prohibited |
| Throw | `RTN_THROW` | Skip this table (policy routing) |

## Route and Address Scopes

Scopes define the reachability of a route or address:

| Scope | Constant | Meaning |
|-------|----------|---------|
| Universe | `RT_SCOPE_UNIVERSE` (0) | Valid everywhere (default for gateways) |
| Site | `RT_SCOPE_SITE` (200) | Interior route (not used in IPv4) |
| Link | `RT_SCOPE_LINK` (253) | Valid on directly attached network |
| Host | `RT_SCOPE_HOST` (254) | Valid only on this host |
| Nowhere | `RT_SCOPE_NOWHERE` (255) | Destination does not exist |

**Scope rules**: A route's scope constrains the scope of its next-hop gateway. A `RT_SCOPE_UNIVERSE` route can have a gateway with `RT_SCOPE_LINK` scope (typical: gateway is on local subnet), but a `RT_SCOPE_LINK` route cannot have a gateway (since it represents a directly connected network).

## Key Data Structures

### `struct rtable`

Routing cache entry. Extends `struct dst_entry` with routing-specific fields:

| Field | Description |
|-------|-------------|
| `u.dst` | Embedded `dst_entry` (next, dev, input/output functions, metrics, neighbour) |
| `rt_flags` | Flags (RTCF_NOTIFY, RTCF_REDIRECTED, RTCF_DOREDIRECT, etc.) |
| `rt_type` | Route type (RTN_UNICAST, RTN_LOCAL, etc.) |
| `rt_dst` | Destination address |
| `rt_src` | Source address (for this route) |
| `rt_gateway` | Next-hop gateway address |
| `rt_spec_dst` | Specific destination (preferred source for replies) |
| `fl` | Flow key this cache entry was created for |
| `peer` | Pointer to `inet_peer` for long-lived per-peer data |
| `rt_iif` | Input interface index |

### `struct dst_entry`

Base destination cache entry (used by both IPv4 and IPv6):

| Field | Description |
|-------|-------------|
| `next` | Hash chain link |
| `__refcnt` | Reference count (atomic) |
| `__use` | Usage counter |
| `dev` | Output device |
| `input` | Input function pointer (`ip_local_deliver`, `ip_forward`, `ip_error`) |
| `output` | Output function pointer (`ip_output`, `ip_mc_output`) |
| `ops` | DST operations (GC, check, destroy callbacks) |
| `metrics[]` | Route metrics (MTU, window, RTT, etc.) indexed by `RTAX_*` |
| `expires` | Expiration timestamp |
| `error` | Error code (for unreachable/prohibited routes) |
| `neighbour` | Associated [neighboring-subsystem](neighboring-subsystem.md) entry |
| `hh` | Cached L2 header |
| `flags` | DST_HOST, DST_NOXFRM, DST_NOPOLICY, etc. |

### `struct flowi`

Flow descriptor used as the routing lookup key:

| Field | Description |
|-------|-------------|
| `fl4_dst` | IPv4 destination address |
| `fl4_src` | IPv4 source address |
| `fl4_tos` | Type of Service |
| `oif` | Output interface index (0 = any) |
| `iif` | Input interface index |
| `proto` | L4 protocol number |
| `fl4_fwmark` | Firewall mark |
| `fl4_scope` | Route scope |
| `uli_u` | Union for L4-specific info (ports, ICMP type/code) |

### `struct fib_result`

Result of a FIB lookup:

| Field | Description |
|-------|-------------|
| `prefix_len` | Length of the matching prefix |
| `nh_sel` | Selected next-hop index (for multipath) |
| `type` | Route type (RTN_UNICAST, etc.) |
| `scope` | Route scope |
| `fi` | Pointer to `fib_info` (route details) |
| `table` | Pointer to the `fib_table` that matched |

## Key Functions

### Input (Ingress) Routing

| Function | Role |
|----------|------|
| `ip_route_input()` | Main entry point. Checks routing cache first; on miss, calls `ip_route_input_slow()` |
| `ip_route_input_slow()` | Performs FIB lookup via `fib_lookup()`, creates `rtable`, inserts into cache via `rt_intern_hash()` |
| `fib_validate_source()` | Reverse path filtering check for ingress packets |

`ip_route_input()` sets `skb->dst` to the matching `rtable`. The `input` function pointer on `dst_entry` determines what happens next:
- `ip_local_deliver` for RTN_LOCAL
- `ip_forward` for RTN_UNICAST (forwarding)
- `ip_error` for RTN_UNREACHABLE/RTN_BLACKHOLE/RTN_PROHIBIT

### Output (Egress) Routing

| Function | Role |
|----------|------|
| `ip_route_output_key()` | Main entry point. Wrapper around `ip_route_output_flow()` |
| `__ip_route_output_key()` | Cache lookup for output routing |
| `ip_route_output_slow()` | Slow path: resolves source address (if not specified), performs FIB lookup, creates `rtable` |

Output routing additionally handles **source address selection**: if the sending socket has not bound to a specific address, `ip_route_output_slow()` selects the best source address based on the outgoing interface and scope rules.

### General

| Function | Role |
|----------|------|
| `fib_lookup()` | Top-level FIB lookup (walks policy routing rules if enabled) |
| `rt_intern_hash()` | Insert `rtable` entry into routing cache hash table |
| `dst_alloc()` | Allocate a `dst_entry` |
| `ip_rt_init()` | Initialize routing cache (hash table, timers, /proc entries) |
| `ip_fib_init()` | Initialize FIB, register netdevice/inetaddr notifiers |

## Routing Subsystem Initialization

At boot time:
1. `ip_rt_init()` allocates the routing cache hash table (sized based on available memory, default goal ~16K entries) and starts GC timers.
2. `ip_fib_init()` creates the default routing tables (`RT_TABLE_LOCAL` and `RT_TABLE_MAIN`), registers notifier callbacks for device and address events.
3. `fib_netdev_event()` and `fib_inetaddr_event()` are registered on `netdev_chain` and `inetaddr_chain` respectively.

## External Events

The routing subsystem reacts to:

| Event | Handler | Action |
|-------|---------|--------|
| `NETDEV_UP` | `fib_netdev_event()` | Add local/broadcast routes for the device |
| `NETDEV_DOWN` | `fib_netdev_event()` | Flush routes using this device, flush routing cache |
| `NETDEV_CHANGE` | `fib_netdev_event()` | Re-validate routes |
| IP addr added | `fib_inetaddr_event()` | Add local route to `RT_TABLE_LOCAL`, flush cache |
| IP addr deleted | `fib_inetaddr_event()` | Remove local route, flush cache |
| Route added/deleted | `rt_cache_flush()` | Invalidate routing cache |

## Interactions with Other Subsystems

- **Neighboring subsystem**: Routing cache entries hold a `neighbour` pointer for L2 address resolution. Set during `rt_intern_hash()` via `arp_bind_neighbour()`.
- **Netfilter**: Hooks at `NF_IP_PRE_ROUTING` (before input route lookup), `NF_IP_FORWARD`, `NF_IP_LOCAL_OUT` (before output route lookup), `NF_IP_POST_ROUTING`.
- **Socket layer**: Sockets cache routing decisions in `sk->sk_dst_cache` to avoid repeated lookups.
- **ICMP**: Routing generates ICMP unreachable/redirect messages. ICMP redirects can modify cached routes.

## Kernel Configuration Options

| Option | Effect |
|--------|--------|
| `CONFIG_IP_ADVANCED_ROUTER` | Enable advanced routing features |
| `CONFIG_IP_MULTIPLE_TABLES` | Enable policy routing (multiple FIB tables + rules) |
| `CONFIG_IP_ROUTE_MULTIPATH` | Enable ECMP/multipath routing |
| `CONFIG_IP_ROUTE_VERBOSE` | Log suspicious routing events |

## Global Locks

| Lock | Protects |
|------|----------|
| `fib_hash_lock` (rwlock) | FIB hash table modifications |
| `fib_info_lock` (rwlock) | `fib_info` list |
| `rt_hash_lock_addr[]` (spinlocks) | Per-bucket routing cache chain locks |
| `fib_rules_lock` (rwlock) | Policy routing rules list |

## See also

- [routing-cache](routing-cache.md)
- [routing-tables-fib](routing-tables-fib.md)
- [neighboring-subsystem](neighboring-subsystem.md)
- [concept-policy-routing](../concepts/concept-policy-routing.md)
- [concept-network-packet-flow](../concepts/concept-network-packet-flow.md)
- [src-understanding-linux-network-internals](../sources/src-understanding-linux-network-internals.md)
