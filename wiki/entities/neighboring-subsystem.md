---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [understanding-linux-network-internals]
tags: [networking, arp, neighbor-discovery, l2-resolution, nud]
---

# Neighboring Subsystem

The neighboring subsystem provides **protocol-independent L3-to-L2 address resolution** in the Linux kernel. It maintains a mapping between network-layer addresses (e.g., IP) and link-layer addresses (e.g., Ethernet MAC), enabling the kernel to fill in the correct destination hardware address before transmitting frames. ARP (IPv4) and Neighbor Discovery (IPv6) are implemented as protocols on top of this common infrastructure.

## Architecture

The subsystem is organized in three layers:

1. **Protocol-independent core** (`net/core/neighbour.c`): State machine, timers, garbage collection, generic neighbor table management.
2. **Protocol-specific handlers**: ARP (`net/ipv4/arp.c`) for IPv4, ND (`net/ipv6/ndisc.c`) for IPv6. Each protocol registers a `neigh_table` with protocol-specific callbacks.
3. **Device-level integration**: Each `net_device` can provide a `neigh_setup()` function and header caching support.

## Key Data Structures

### `struct neigh_table`

Per-protocol neighbor table (one for ARP, one for ND). Key fields:

| Field | Description |
|-------|-------------|
| `family` | Address family (AF_INET, AF_INET6) |
| `entry_size` | Size of each neighbor entry |
| `key_len` | Length of the L3 address key |
| `hash` | Hash function for lookup |
| `constructor` | Protocol-specific neighbor initialization |
| `proxy_redo` | Function to re-process proxied solicitations |
| `id` | Table name string |
| `parms` | Default `neigh_parms` for the table |
| `gc_interval` | Garbage collection interval |
| `gc_thresh1/2/3` | GC thresholds (soft, medium, hard limits) |
| `hash_buckets` | Hash table for neighbor entries |
| `hash_mask` | Hash table size mask |
| `phash_buckets` | Hash table for proxy entries |
| `entries` | Current number of entries |
| `last_flush` | Timestamp of last forced flush |
| `gc_timer` | Timer for periodic GC |
| `proxy_timer` | Timer for proxy queue processing |
| `proxy_queue` | Queue of proxied solicitations |

### `struct neighbour`

Individual neighbor entry representing one L3-to-L2 mapping. Key fields:

| Field | Description |
|-------|-------------|
| `next` | Hash chain link |
| `tbl` | Back pointer to owning `neigh_table` |
| `parms` | Per-interface `neigh_parms` |
| `dev` | Associated `net_device` |
| `used` | Timestamp of last use |
| `confirmed` | Timestamp of last reachability confirmation |
| `updated` | Timestamp of last state change |
| `nud_state` | Current NUD state (see below) |
| `type` | Address type |
| `dead` | Entry marked for deletion |
| `refcnt` | Reference count (atomic) |
| `lock` | Read-write lock |
| `ha[]` | Hardware (L2) address |
| `primary_key[]` | L3 address (variable-length) |
| `timer` | NUD state machine timer |
| `arp_queue` | Queue of packets waiting for resolution |
| `ops` | Pointer to `neigh_ops` |
| `output` | Current output function pointer |

### `struct neigh_ops`

Function pointer table for neighbor operations, set based on NUD state:

| Function | Description |
|----------|-------------|
| `family` | Address family |
| `solicit` | Send solicitation (e.g., ARP request) |
| `error_report` | Report error to upper layer |
| `output` | Generic output (used when state is not confirmed) |
| `connected_output` | Fast output (used when state is NUD_CONNECTED) |
| `hh_output` | Fastest output (used when L2 header is cached) |
| `queue_xmit` | Raw device transmit |

### `struct neigh_parms`

Per-interface tunable parameters. Key fields:

| Field | Description |
|-------|-------------|
| `base_reachable_time` | Base time a neighbor is considered reachable (default 30s) |
| `retrans_time` | Retransmission interval for solicitations |
| `gc_staletime` | Time before stale entry eligible for GC |
| `reachable_time` | Computed reachable time (randomized from base) |
| `delay_probe_time` | Delay before probing a stale entry |
| `ucast_probes` | Max unicast probes before failure |
| `mcast_probes` | Max multicast/broadcast probes before failure |
| `app_probes` | Max probes sent to user-space daemon |
| `proxy_delay` | Max random delay for proxy responses |
| `proxy_qlen` | Max proxy queue length |
| `unres_qlen` | Max unresolved queue length per entry |
| `locktime` | Min time between neighbor updates |

## NUD State Machine

The Network Unreachability Detection (NUD) state machine tracks neighbor reachability:

```
                    +----> NUD_PERMANENT (statically configured)
                    |
NUD_NONE --solicit--> NUD_INCOMPLETE --response--> NUD_REACHABLE
                         |                              |
                         | timeout                      | timeout (base_reachable_time)
                         v                              v
                    NUD_FAILED                     NUD_STALE
                                                       |
                                                       | traffic sent
                                                       v
                                                  NUD_DELAY
                                                       |
                                                       | delay_probe_time expires
                                                       v
                                                  NUD_PROBE
                                                    /     \
                                              reply /       \ max probes
                                                  v         v
                                           NUD_REACHABLE  NUD_FAILED
```

State groups:
- **NUD_VALID**: REACHABLE, STALE, DELAY, PROBE, PERMANENT, NOARP -- entry has a valid L2 address
- **NUD_CONNECTED**: REACHABLE, PERMANENT, NOARP -- entry is confirmed reachable, use fast `connected_output`
- **NUD_FAILED/NUD_INCOMPLETE**: Resolution not (yet) completed

The `output` function pointer on `struct neighbour` switches dynamically based on NUD state: `connected_output` when connected (fastest), `output` otherwise (may trigger re-resolution).

## Key Functions

| Function | Role |
|----------|------|
| `neigh_lookup()` | Find existing neighbor entry by L3 address |
| `neigh_create()` | Allocate and initialize new neighbor entry |
| `neigh_resolve_output()` | Resolve address and transmit (may queue packet) |
| `neigh_connected_output()` | Fast-path transmit for confirmed neighbors |
| `neigh_event_ns()` | Process incoming solicitation |
| `neigh_update()` | Update neighbor state/address (main state transition function) |
| `neigh_timer_handler()` | NUD timer -- drives state transitions |
| `neigh_periodic_timer()` | Periodic GC -- removes stale/failed entries |
| `neigh_forced_gc()` | Synchronous GC when table is full |
| `neigh_delete()` | Remove a neighbor entry |
| `neigh_alloc()` | Low-level allocation |
| `pneigh_lookup()` | Look up or create proxy neighbor entry |
| `neigh_proxy_process()` | Process queued proxy solicitations |

## L2 Header Caching

For frequently used neighbors, the subsystem caches the complete L2 header in a `struct hh_cache` to avoid rebuilding it on every packet:

- `neigh_hh_init()`: Creates a cached header by calling `dev->hard_header()` with the known hardware address
- When cached, packets use `neigh_hh_output()` which copies the pre-built header directly, avoiding per-packet `hard_header()` calls
- Cache is invalidated when the neighbor's hardware address changes

## Garbage Collection

Two GC mechanisms keep the neighbor table bounded:

1. **Periodic GC** (`neigh_periodic_timer()`): Runs every `gc_interval` (default 30s). Removes entries in NUD_FAILED state and entries in NUD_STALE that exceed `gc_staletime`. Respects `gc_thresh1` (below this, no GC).

2. **Forced GC** (`neigh_forced_gc()`): Triggered when entry count exceeds `gc_thresh2` (with table recently grown) or `gc_thresh3` (hard limit). Aggressively removes non-permanent entries.

Thresholds (defaults for ARP):
- `gc_thresh1` = 128 (no GC below this)
- `gc_thresh2` = 512 (trigger periodic GC)
- `gc_thresh3` = 1024 (hard limit, force GC)

## Interaction with Other Subsystems

- **Routing**: When `ip_route_input()` or `ip_route_output()` creates a routing cache entry, it binds a `neighbour` to the `dst_entry->neighbour` field. All subsequent transmissions via that route use the bound neighbor for L2 resolution.
- **Device layer**: `NETDEV_DOWN` notification triggers `neigh_ifdown()`, which flushes all neighbor entries for that device.
- **L3 protocols**: The neighboring subsystem is invoked at the boundary between L3 and L2 during transmission. `dst->neighbour->output()` is the handoff point.

## /proc Tuning

Parameters under `/proc/sys/net/ipv4/neigh/<interface>/`:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `base_reachable_time` | 30 | Base reachable time (seconds) |
| `retrans_time` | 1 | Retransmission interval (seconds) |
| `gc_stale_time` | 60 | Time before stale entry is GC-eligible |
| `unres_qlen` | 3 | Max queued packets per unresolved entry |
| `proxy_qlen` | 64 | Max queued proxy solicitations |
| `app_solicit` | 0 | Solicitations sent to user-space daemon |
| `ucast_solicit` | 3 | Max unicast solicitations |
| `mcast_solicit` | 3 | Max broadcast solicitations |
| `delay_first_probe_time` | 5 | Delay before first probe of stale entry |
| `locktime` | 1 | Min interval between updates |

## See also

- [arp](arp.md)
- [routing-subsystem](routing-subsystem.md)
- [net-device](net-device.md)
- [concept-network-packet-flow](../concepts/concept-network-packet-flow.md)
- [src-understanding-linux-network-internals](../sources/src-understanding-linux-network-internals.md)
