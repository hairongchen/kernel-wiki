---
type: concept
created: 2026-04-09
updated: 2026-04-09
sources: [understanding-linux-network-internals]
tags: [networking, routing, policy-routing, fib, ipv4]
---

# Policy Routing

Policy routing extends Linux's routing subsystem beyond simple destination-based forwarding, allowing routing decisions based on **source address, TOS, firewall mark, and input interface** in addition to destination. Enabled by `CONFIG_IP_MULTIPLE_TABLES`, it replaces the single-table FIB lookup with a **rule-based system** that selects among multiple routing tables.

## Motivation

Standard destination-based routing uses a single routing table and always makes the same forwarding decision for packets going to the same destination. Policy routing is needed when:

- **Multi-homed hosts**: Different source subnets should use different ISP uplinks
- **Quality of Service**: Different TOS values should take different paths
- **VPN / overlay routing**: Traffic marked by iptables (`fwmark`) should be routed through tunnels
- **Source-based routing**: Replies should leave through the same interface where requests arrived

## Architecture

```
                        fib_lookup(flowi)
                             |
                             v
                   +-----------------+
                   | fib_rules list  |  (ordered by r_preference)
                   +-----------------+
                   | Rule 0: -> LOCAL(255)    |  always checked first
                   | Rule 100: src 10/8 -> T100 |
                   | Rule 200: fwmark 1 -> T200 |
                   | Rule 32766: -> MAIN(254)   |
                   | Rule 32767: UNREACHABLE     |
                   +-----------------------------+
                        |         |         |
                        v         v         v
                  fib_table    fib_table   fib_table
                  (LOCAL)      (100)       (MAIN)
```

Without `CONFIG_IP_MULTIPLE_TABLES`, `fib_lookup()` is a simple inline function that looks up `RT_TABLE_LOCAL` first, then `RT_TABLE_MAIN`. With it enabled, `fib_lookup()` walks the rules list, evaluating each rule's selectors against the flow.

## Rule Matching

Each `struct fib_rule` has selectors that are ANDed together. A rule matches if **all** of its non-zero selectors match the packet:

| Selector | Field | Matches against |
|----------|-------|-----------------|
| Source address | `r_src` / `r_srcmask` | `flowi.fl4_src` |
| Destination address | `r_dst` / `r_dstmask` | `flowi.fl4_dst` |
| TOS | `r_tos` | `flowi.fl4_tos` |
| Firewall mark | `r_fwmark` | `flowi.fl4_fwmark` |
| Input interface | `r_ifindex` | `flowi.iif` |

A selector value of 0 means "match any" for that field.

## Rule Actions

When a rule matches, its `r_action` determines what happens:

| Action | Constant | Behavior |
|--------|----------|----------|
| Look up table | `FR_ACT_TO_TBL` | Perform FIB lookup in `r_table`. If found (and not RTN_THROW), return result. If RTN_THROW or not found, continue to next rule. |
| Unreachable | `FR_ACT_UNREACHABLE` | Return `ENETUNREACH` (generates ICMP destination unreachable) |
| Prohibit | `FR_ACT_PROHIBIT` | Return `EACCES` (generates ICMP communication administratively prohibited) |
| Blackhole | `FR_ACT_BLACKHOLE` | Silently drop the packet |

The `RTN_THROW` route type is specifically designed for policy routing: when a lookup matches a "throw" route, the kernel **continues to the next rule** rather than using that result. This allows a table to explicitly decline routing certain destinations, falling through to less specific rules.

## Default Rule Configuration

Three rules are created at boot:

| Priority | Selector | Action | Table |
|----------|----------|--------|-------|
| 0 | Match all | `FR_ACT_TO_TBL` | `RT_TABLE_LOCAL` (255) |
| 32766 | Match all | `FR_ACT_TO_TBL` | `RT_TABLE_MAIN` (254) |
| 32767 | Match all | `FR_ACT_UNREACHABLE` | -- |

`RT_TABLE_LOCAL` is special: the kernel automatically populates it with local and broadcast addresses. It is always checked first (priority 0) so that locally-addressed packets are always delivered.

## Routing Tables

Up to 256 routing tables (IDs 0-255) can exist:

| ID | Name | Purpose |
|----|------|---------|
| 0 | `RT_TABLE_UNSPEC` | Unspecified (not used) |
| 253 | `RT_TABLE_DEFAULT` | Default route table |
| 254 | `RT_TABLE_MAIN` | Main routing table (where `ip route add` goes by default) |
| 255 | `RT_TABLE_LOCAL` | Local addresses (kernel-managed) |
| 1-252 | User-defined | Custom tables for policy routing |

Tables are allocated on demand by `fib_new_table()` and stored in an array indexed by table ID.

## Interaction with the Routing Cache

Policy routing complicates caching because the same destination may be routed differently depending on source, TOS, or fwmark:

- The [routing-cache](../entities/routing-cache.md) includes **all flow key fields** (src, dst, tos, fwmark, iif/oif) in both the hash function and the match comparison
- This means each unique policy-routing-relevant flow gets its own cache entry
- Cache entries store which `fib_table` produced the result, enabling proper invalidation

## Interaction with Reverse Path Filtering

`fib_validate_source()` performs a reverse lookup to check that the source address of an incoming packet would be routed back through the receiving interface. With policy routing, this reverse lookup also walks the rules, which can lead to asymmetric routing being falsely flagged. The `rp_filter` setting can be set to:
- 0: No reverse path filtering
- 1: Strict mode (must route back through same interface)
- 2: Loose mode (source must be routable through some interface)

## Practical Examples

```bash
# Scenario: Two ISPs, route based on source subnet
# ISP1 via 10.0.1.1, subnet 10.0.1.0/24
# ISP2 via 10.0.2.1, subnet 10.0.2.0/24

# Create custom tables
ip route add default via 10.0.1.1 table 100
ip route add default via 10.0.2.1 table 200

# Add rules
ip rule add from 10.0.1.0/24 table 100 priority 100
ip rule add from 10.0.2.0/24 table 200 priority 200

# Scenario: Route marked traffic through VPN
iptables -t mangle -A OUTPUT -p tcp --dport 443 -j MARK --set-mark 1
ip route add default via 10.8.0.1 table 300
ip rule add fwmark 1 table 300 priority 150
```

## Implementation Details

- Rules are stored in a global linked list `fib_rules` protected by `fib_rules_lock` (rwlock)
- `fib_lookup()` walks the list under read lock for every cache miss
- Rules are ordered by `r_preference` (lower = higher priority)
- `ip rule add` / `ip rule del` use netlink messages `RTM_NEWRULE` / `RTM_DELRULE`
- Adding or deleting a rule triggers `rt_cache_flush()` to invalidate cached routes that may now follow different rules

## See also

- [routing-subsystem](../entities/routing-subsystem.md)
- [routing-tables-fib](../entities/routing-tables-fib.md)
- [routing-cache](../entities/routing-cache.md)
- [src-understanding-linux-network-internals](../sources/src-understanding-linux-network-internals.md)
