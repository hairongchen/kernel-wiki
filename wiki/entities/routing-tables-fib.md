---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [understanding-linux-network-internals]
tags: [networking, routing, fib, ipv4, forwarding, longest-prefix-match]
---

# Routing Tables (FIB)

The Forwarding Information Base (FIB) is the **slow-path** component of the Linux [routing-subsystem](routing-subsystem.md). It stores the routing table entries configured by administrators and routing daemons, and performs **longest-prefix-match** lookups when the [routing-cache](routing-cache.md) has a miss. The FIB is implemented with a pluggable backend architecture, with two available implementations: `fn_hash` (hash-based, default) and `fn_trie` (LC-trie, optimized for large tables).

## Architecture

```
                   fib_lookup()
                       |
                       v
             [Policy Routing Rules]
             (if CONFIG_IP_MULTIPLE_TABLES)
                       |
          selects table (RT_TABLE_LOCAL, RT_TABLE_MAIN, ...)
                       |
                       v
               struct fib_table
              /                \
             /                  \
     fn_hash backend        fn_trie backend
     (hash per prefix       (compressed trie /
      length, 0-32)          LC-trie)
```

## Key Data Structures

### `struct fib_table`

Represents one routing table. Uses function pointers for polymorphism across backends:

| Field | Description |
|-------|-------------|
| `tb_id` | Table ID (RT_TABLE_LOCAL=255, RT_TABLE_MAIN=254, RT_TABLE_DEFAULT=253, or user-defined 1-252) |
| `tb_stamp` | Last modification timestamp |
| `tb_lookup()` | Backend-specific lookup function |
| `tb_insert()` | Backend-specific route insertion |
| `tb_delete()` | Backend-specific route deletion |
| `tb_flush()` | Backend-specific bulk flush |
| `tb_select_default()` | Backend-specific default route selection |
| `tb_dump()` | Dump table contents (for netlink) |
| `tb_data[]` | Backend-specific data (flexible array) |

### `struct fib_info`

Shared route information. Multiple routes with different prefixes can share a `fib_info` if they have the same attributes (gateway, metrics, protocol, etc.):

| Field | Description |
|-------|-------------|
| `fib_hash` | Link in global `fib_info_hash` for deduplication |
| `fib_treeref` | Number of `fib_node` entries referencing this |
| `fib_clntref` | Client reference count (from `rtable` entries) |
| `fib_dead` | Marked for deletion |
| `fib_flags` | Route flags (RTNH_F_DEAD, RTNH_F_PERVASIVE, RTNH_F_ONLINK) |
| `fib_protocol` | Route source: RTPROT_KERNEL, RTPROT_BOOT, RTPROT_STATIC, RTPROT_REDIRECT, or routing daemon protocol ID |
| `fib_scope` | Route scope (RT_SCOPE_UNIVERSE, etc.) |
| `fib_type` | Route type (RTN_UNICAST, RTN_LOCAL, etc.) |
| `fib_prefsrc` | Preferred source address for this route |
| `fib_priority` | Route priority/metric (lower = preferred) |
| `fib_metrics[]` | Array of route metrics indexed by RTAX_* constants (MTU, window, RTT, etc.) |
| `fib_nhs` | Number of next hops |
| `fib_power` | Remaining multipath power (sum of all next-hop weights) |
| `fib_nh[]` | Array of `fib_nh` next-hop structures (flexible array) |

### `struct fib_nh`

Represents one next hop in a route:

| Field | Description |
|-------|-------------|
| `nh_dev` | Output `net_device` |
| `nh_hash` | Hash chain link |
| `nh_parent` | Back pointer to owning `fib_info` |
| `nh_flags` | Flags (RTNH_F_DEAD, RTNH_F_PERVASIVE, RTNH_F_ONLINK) |
| `nh_scope` | Scope of the gateway address |
| `nh_weight` | Multipath weight (for load balancing) |
| `nh_power` | Remaining power budget (decremented on use, reset when all zero) |
| `nh_oif` | Output interface index |
| `nh_gw` | Gateway (next-hop) IP address |
| `nh_tclassid` | Traffic control classifier ID |

### `struct fib_node`

Represents a single destination prefix in the FIB:

| Field | Description |
|-------|-------------|
| `fn_hash` | Link in `fn_zone` hash chain |
| `fn_key` | Destination network address (masked to prefix length) |
| `fn_alias` | List of `fib_alias` entries for this prefix |

### `struct fib_alias`

One route variant for a given prefix. Multiple aliases exist when routes with the same prefix differ in TOS, type, or scope:

| Field | Description |
|-------|-------------|
| `fa_list` | Link in `fib_node.fn_alias` list |
| `fa_info` | Pointer to shared `fib_info` |
| `fa_tos` | TOS value (0 for no TOS matching) |
| `fa_type` | Route type (RTN_UNICAST, etc.) |
| `fa_scope` | Route scope |
| `fa_state` | State flags (FA_S_ACCESSED) |

### Relationship Diagram

```
fib_table
    |
    tb_data --> [fn_zone array, 0..32]    (fn_hash backend)
                     |
                fn_zone[prefix_len]
                     |
                  hash table --> fib_node (fn_key = network addr)
                                    |
                                 fib_alias --> fib_info --> fib_nh[]
                                 fib_alias --> fib_info --> fib_nh[]
                                    (one per TOS/type/scope variant)
```

## fn_hash Backend

The default FIB backend. Organizes routes by prefix length:

### `struct fn_zone`

One zone per prefix length (0 through 32), but only allocated for prefix lengths that have routes:

| Field | Description |
|-------|-------------|
| `fz_hash` | Hash table of `fib_node` entries |
| `fz_nent` | Number of entries in this zone |
| `fz_divisor` | Hash table size |
| `fz_hashmask` | Hash table mask |
| `fz_order` | Prefix length (0-32) |
| `fz_mask` | Network mask for this prefix length |
| `fz_next` | Link to next zone (ordered by decreasing prefix length) |

### Lookup: `fn_hash_lookup()`

1. Walk zones from longest prefix (32) to shortest (0) -- this implements longest prefix match.
2. For each zone: hash the destination masked to the zone's prefix length.
3. Walk the hash chain, comparing `fib_node.fn_key`.
4. For each matching `fib_node`: walk `fib_alias` list, checking TOS and scope.
5. First match wins (aliases are ordered by TOS descending, then by scope).

Time complexity: O(33) worst case (one hash lookup per prefix length), but typically much less since most zones are empty.

## fn_trie Backend (LC-Trie)

An alternative backend using Level-Compressed tries, enabled with `CONFIG_IP_ROUTE_FIB_TRIE`. Optimized for large routing tables (BGP full table with ~150K+ routes).

### Structure

- **Internal nodes (`tnode`)**: Have a prefix, position, and number of child bits. Children are pointers to other tnodes or leaves.
- **Leaf nodes**: Contain `fib_alias` lists, same as fn_hash.
- **Path compression**: Nodes with a single child are compressed (the "level-compressed" part).

### Lookup: `fn_trie_lookup()`

Walks the trie from root, using bits of the destination address to select children. On reaching a leaf, checks the `fib_alias` list for a match. Backtracks to shorter prefixes if no match. Time complexity: O(W) where W = address width (32 bits for IPv4).

### Tradeoffs

| Aspect | fn_hash | fn_trie |
|--------|---------|---------|
| Lookup complexity | O(33) hash probes worst case | O(32) trie levels worst case |
| Memory | Hash tables per zone | Trie nodes |
| Best for | Small-medium tables (<10K routes) | Large tables (full BGP, 100K+ routes) |
| Insert/delete | O(1) per zone | O(W) trie update |
| Default | Yes | No (requires CONFIG_IP_ROUTE_FIB_TRIE) |

## Route Addition Flow

When a route is added (e.g., `ip route add 10.0.0.0/8 via 192.168.1.1`):

1. **Netlink message** received by `inet_rtm_newroute()`.
2. **`fib_create_info()`**: Allocates and initializes `fib_info` with next-hop(s). Validates gateway reachability. Checks for duplicate `fib_info` in global hash (shares if identical). Resolves next-hop device from gateway address.
3. **`fib_table_insert()` / `fn_hash_insert()`**:
   a. Find or create `fn_zone` for the prefix length.
   b. Find or create `fib_node` for the destination network.
   c. Create `fib_alias` with the TOS/type/scope.
   d. Insert alias into the node's alias list (ordered by TOS, then scope).
   e. Handle `NLM_F_REPLACE` (replace existing) vs `NLM_F_APPEND` (add even if duplicate).
4. **`rt_cache_flush()`**: Invalidate routing cache since tables changed.
5. **Send netlink notification** to user-space listeners.

## Route Deletion Flow

1. **`inet_rtm_delroute()`** receives netlink message.
2. **`fib_table_delete()` / `fn_hash_delete()`**:
   a. Find `fib_node` by destination + prefix length.
   b. Find matching `fib_alias` by TOS/type/scope.
   c. Remove alias from list.
   d. If last alias: remove `fib_node`, potentially remove `fn_zone`.
   e. Call `fib_release_info()` on the `fib_info`.
3. **`rt_cache_flush()`**: Invalidate routing cache.

## Default Route Selection: `fib_select_default()`

When `fib_lookup()` returns a default route (prefix 0.0.0.0/0) and multiple default routes exist:

1. Walk all `fib_alias` entries for the default prefix.
2. Prefer routes that are:
   - Not dead (`RTNH_F_DEAD` not set)
   - Associated with a live device
   - Added by a routing protocol (RTPROT_REDIRECT) over static
3. Use `fib_info.fib_priority` to break ties (lower priority value = preferred).
4. Implements round-robin among equal-priority default routes.

## Policy Routing

When `CONFIG_IP_MULTIPLE_TABLES` is enabled, the FIB supports multiple routing tables selected by rules.

### `struct fib_rule`

| Field | Description |
|-------|-------------|
| `r_preference` | Rule priority (lower = checked first) |
| `r_table` | Which `fib_table` to use for matching traffic |
| `r_action` | Action: FR_ACT_TO_TBL, FR_ACT_UNREACHABLE, FR_ACT_PROHIBIT, FR_ACT_BLACKHOLE |
| `r_src` / `r_srcmask` | Source address/mask selector |
| `r_dst` / `r_dstmask` | Destination address/mask selector |
| `r_tos` | TOS selector |
| `r_fwmark` | Firewall mark selector |
| `r_ifindex` | Input interface selector |

### Default Rules

Three rules exist by default:
1. **Priority 0**: Match all, lookup `RT_TABLE_LOCAL` (255) -- local/broadcast addresses
2. **Priority 32766**: Match all, lookup `RT_TABLE_MAIN` (254) -- normal routes
3. **Priority 32767**: Match all, action `FR_ACT_UNREACHABLE` -- final fallback

### `fib_lookup()` with Policy Routing

```
for each rule (ordered by r_preference):
    if packet matches rule's selectors (src, dst, tos, fwmark, iif):
        if rule.r_action == FR_ACT_TO_TBL:
            result = rule.r_table->tb_lookup(...)
            if result found and result.type != RTN_THROW:
                return result
            else:
                continue to next rule
        elif rule.r_action in {FR_ACT_UNREACHABLE, FR_ACT_PROHIBIT, FR_ACT_BLACKHOLE}:
            return corresponding error
return -ENETUNREACH
```

### Administration

```bash
# View policy rules
ip rule list

# Add rule: traffic from 10.0.0.0/8 uses table 100
ip rule add from 10.0.0.0/8 table 100

# Add rule: traffic with fwmark 1 uses table 200
ip rule add fwmark 1 table 200

# View specific table
ip route show table 100

# Add route to custom table
ip route add default via 10.0.0.1 table 100
```

## `fib_info` Lifecycle and Sharing

`fib_info` structures are **deduplicated** via a global hash table (`fib_info_hash`):

1. When creating a new route, `fib_create_info()` computes a hash of the route attributes.
2. Searches `fib_info_hash` for an identical existing `fib_info`.
3. If found: increments `fib_treeref`, returns the existing one.
4. If not: allocates new, inserts into hash.
5. `fib_release_info()` decrements `fib_treeref`; when zero, frees the `fib_info`.

This sharing is critical for memory efficiency: many routes (e.g., all routes via the same gateway) can share a single `fib_info`.

## See also

- [routing-subsystem](routing-subsystem.md)
- [routing-cache](routing-cache.md)
- [concept-policy-routing](../concepts/concept-policy-routing.md)
- [neighboring-subsystem](neighboring-subsystem.md)
- [src-understanding-linux-network-internals](../sources/src-understanding-linux-network-internals.md)
