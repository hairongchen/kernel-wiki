---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [understanding-linux-network-internals]
tags: [networking, routing, cache, ipv4, performance]
---

# Routing Cache

The routing cache is the **fast-path** of the Linux [routing-subsystem](routing-subsystem.md). It stores the results of previous route lookups in a hash table of `struct rtable` entries, allowing subsequent packets in the same flow to bypass the more expensive FIB (Forwarding Information Base) lookup. The cache is implemented in `net/ipv4/route.c`.

## Hash Table Organization

The routing cache uses a hash table `rt_hash_table[]` of `struct rt_hash_bucket` entries:

```c
struct rt_hash_bucket {
    struct rtable *chain;   // Singly-linked list of rtable entries
};
```

- **Size**: Determined at boot by `ip_rt_init()` based on available memory. Typical sizes range from 2K to 256K buckets. Rounded to power of 2 for fast masking.
- **Hash function**: `rt_hash(daddr, saddr, iif_or_oif)` -- combines destination, source, and interface index. Uses Jenkins hash or similar mixing function with a random secret (`rt_hash_rnd`) that is periodically regenerated.
- **Locking**: Per-bucket spinlocks in `rt_hash_lock_addr[]`. Lookups are lock-free using RCU; insertions/deletions take the bucket lock.

## Cache Lookup

### Input Routing: `ip_route_input()`

Called for every incoming packet (from `ip_rcv_finish()`). Fast-path algorithm:

1. Compute hash from `(daddr, saddr, iif)`.
2. Walk the hash chain under RCU read lock.
3. For each `rtable`, compare: `rt_dst == daddr && rt_src == saddr && rt_iif == iif && rt_tos == tos && rt_mark == fwmark`.
4. On match: increment `dst.__use`, set `skb->dst = &rt->u.dst`, return 0.
5. On miss: call `ip_route_input_slow()` for FIB lookup.

### Output Routing: `__ip_route_output_key()`

Called when a socket wants to send. Fast-path algorithm:

1. Compute hash from `(daddr, saddr, oif)`.
2. Walk the hash chain under RCU read lock.
3. Match against flow key fields: `fl.fl4_dst`, `fl.fl4_src`, `fl.oif`, `fl.fl4_tos`, `fl.fl4_fwmark`.
4. On match: increment `dst.__use`, return the `rtable`.
5. On miss: call `ip_route_output_slow()`.

## Cache Entry Creation

When a cache miss occurs, the slow-path functions create a new `rtable`:

1. **`dst_alloc()`**: Allocate a `dst_entry` from the `ipv4_dst_ops` slab cache.
2. **Fill fields**: Set `rt_dst`, `rt_src`, `rt_gateway`, `rt_type`, `rt_flags`, `fl` (flow key), `input`/`output` function pointers based on FIB result.
3. **Bind neighbor**: Call `arp_bind_neighbour()` to associate a [neighboring-subsystem](neighboring-subsystem.md) entry for the next-hop gateway.
4. **Bind peer**: Call `rt_bind_peer()` to find or create an `inet_peer` for long-lived per-destination data.
5. **`rt_intern_hash()`**: Insert into the routing cache hash table.

### `rt_intern_hash()`

The insertion function performs several checks:

1. **Duplicate detection**: Walk the chain to check if an equivalent entry already exists. If found, use the existing one and free the new one.
2. **Chain length check**: If the chain exceeds a threshold, trigger garbage collection.
3. **Aging**: Set `dst.expires` and `dst.__use`.
4. **Insert at head**: Prepend to the chain (new entries are "hotter").

## Multipath Caching

With multipath routing (CONFIG_IP_ROUTE_MULTIPATH), multiple next hops exist for a single destination. The routing cache handles this via:

- **`fib_select_multipath()`**: Called during slow-path lookup. Selects a next hop based on the configured algorithm (weighted random, round-robin, etc.).
- **Per-flow caching**: Each unique flow (src+dst+tos+fwmark combination) gets its own cache entry with its selected next hop. This provides per-flow load balancing while keeping all packets of a single flow on the same path.
- **Power-based selection**: Each `fib_nh` has `nh_weight` (configured) and `nh_power` (remaining budget). When `nh_power` reaches 0, that next hop is skipped. The parent `fib_info.fib_power` tracks the total remaining budget.

## Interface with DST Subsystem

The routing cache integrates with the generic destination cache (`dst_entry`) framework:

| DST Operation | Routing Cache Implementation |
|---------------|------------------------------|
| `dst->output()` | `ip_output()`, `ip_mc_output()`, or `ip_rt_bug()` |
| `dst->input()` | `ip_local_deliver()`, `ip_forward()`, or `ip_error()` |
| `dst_ops->gc()` | `rt_garbage_collect()` |
| `dst_ops->check()` | Validates entry is still current |
| `dst_ops->destroy()` | Releases `inet_peer` and neighbor references |
| `dst_ops->negative_advice()` | Called when transport layer detects issues; may remove cached redirect |

The DST framework provides reference counting (`dst_hold()` / `dst_release()`), ensuring entries are not freed while in use by sockets or in-flight packets.

## Garbage Collection

Three mechanisms keep the cache bounded:

### 1. Asynchronous GC: `rt_check_expire()`

- Runs periodically via timer (default interval: `ip_rt_gc_interval`, typically 60 seconds)
- Walks a portion of the hash table per invocation (avoiding a full scan each time)
- Removes entries where `jiffies > dst.expires` (entry has timed out)
- Respects `ip_rt_gc_timeout` (minimum age before eligible for GC, default 300 seconds)
- Also checks `dst.__use` -- frequently used entries get extended lifetime

### 2. Synchronous GC: `rt_garbage_collect()`

- Triggered when cache is full (entry count exceeds `ip_rt_max_size`) during `rt_intern_hash()`
- Much more aggressive than async GC
- Uses progressively shorter expiry thresholds:
  - First pass: remove entries older than `ip_rt_gc_elasticity * ip_rt_gc_timeout`
  - Subsequent passes: halve the threshold
  - Last resort: remove entries older than 1 jiffy
- Uses a **goal-based approach**: tries to free at least `ip_rt_gc_min_interval` entries
- Tracks equilibrium state to avoid thrashing

### 3. Cache Flush: `rt_cache_flush()` / `rt_run_flush()`

- Called when routing tables change (routes added/deleted, interfaces up/down)
- Does NOT immediately purge entries -- instead sets a flush timer
- `rt_run_flush()` walks the entire hash table, moving all entries to a "flush" list
- Entries on the flush list are freed when their reference count drops to zero
- Multiple rapid changes are coalesced by the flush timer (avoids thrashing)

### Tuning Parameters

| Parameter | /proc path | Default | Description |
|-----------|-----------|---------|-------------|
| `gc_timeout` | `/proc/sys/net/ipv4/route/gc_timeout` | 300 | Min age (seconds) for async GC eligibility |
| `gc_interval` | `/proc/sys/net/ipv4/route/gc_interval` | 60 | Async GC timer period |
| `gc_min_interval` | `/proc/sys/net/ipv4/route/gc_min_interval` | 0 | Min interval between sync GC runs |
| `gc_elasticity` | `/proc/sys/net/ipv4/route/gc_elasticity` | 8 | Multiplier for sync GC threshold |
| `max_size` | `/proc/sys/net/ipv4/route/max_size` | varies | Max number of cache entries |
| `max_delay` | `/proc/sys/net/ipv4/route/flush` | 10 | Max flush delay (seconds). Writing to this file triggers immediate flush |
| `min_delay` | (internal) | 2 | Min flush delay |
| `redirect_number` | `/proc/sys/net/ipv4/route/redirect_number` | 9 | Max ICMP redirects sent per destination |
| `redirect_silence` | `/proc/sys/net/ipv4/route/redirect_silence` | varies | Silence period after redirect_number is reached |

## ICMP Redirect Handling

### Incoming Redirects: `ip_rt_redirect()`

When the kernel receives an ICMP redirect:

1. Validate: the redirect must come from the current gateway for the destination.
2. Look up the suggested new gateway -- it must be on a directly connected network.
3. If valid: find the existing `rtable` entry, create a new one with `RTCF_REDIRECTED` flag and the new gateway, insert via `rt_intern_hash()`.
4. Redirected entries are marked and can be removed by `dst_ops->negative_advice()` if the new route proves problematic.

### Outgoing Redirects: `ip_rt_send_redirect()`

When forwarding a packet and the next hop is on the same interface as the source:

1. The `rtable` is flagged with `RTCF_DOREDIRECT`.
2. `ip_rt_send_redirect()` implements **token-bucket rate limiting**:
   - Tracks per-peer redirect count and timestamps via `inet_peer`
   - Exponential backoff: first redirect sent immediately, subsequent ones with doubling delay
   - Maximum `ip_rt_redirect_number` redirects per destination
   - After limit: enters silence period (`ip_rt_redirect_silence`)

## /proc Interfaces

- `/proc/net/rt_cache`: Dumps all routing cache entries (destination, gateway, source, flags, use count, interface, etc.)
- `/proc/sys/net/ipv4/route/*`: Tuning parameters (see table above)
- `/proc/net/stat/rt_cache`: Cache statistics (lookups, hits, misses, GC runs, etc.)

## See also

- [routing-subsystem](routing-subsystem.md)
- [routing-tables-fib](routing-tables-fib.md)
- [neighboring-subsystem](neighboring-subsystem.md)
- [concept-policy-routing](../concepts/concept-policy-routing.md)
- [src-understanding-linux-network-internals](../sources/src-understanding-linux-network-internals.md)
