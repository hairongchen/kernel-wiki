---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [understanding-linux-network-internals]
tags: [networking, arp, neighbor-discovery, ipv4, l2-resolution]
---

# Address Resolution Protocol (ARP)

ARP is IPv4's implementation of the [neighboring-subsystem](neighboring-subsystem.md). It maps IPv4 addresses to Ethernet MAC addresses, enabling IP packets to be encapsulated in Ethernet frames with the correct destination hardware address. Defined in RFC 826, the Linux implementation lives in `net/ipv4/arp.c` and builds on the protocol-independent neighboring infrastructure.

## ARP Packet Format

```
struct arphdr {
    __be16 ar_hrd;    // Hardware type (ARPHRD_ETHER = 1)
    __be16 ar_pro;    // Protocol type (ETH_P_IP = 0x0800)
    __u8   ar_hln;    // Hardware address length (6 for Ethernet)
    __u8   ar_pln;    // Protocol address length (4 for IPv4)
    __be16 ar_op;     // Operation: ARPOP_REQUEST=1, ARPOP_REPLY=2
};
// Followed by variable-length fields:
//   sender hardware address (ar_hln bytes)
//   sender protocol address (ar_pln bytes)
//   target hardware address (ar_hln bytes)
//   target protocol address (ar_pln bytes)
```

The kernel accesses the variable-length fields via `arp_ptr` pointer arithmetic starting after the fixed header, using `ar_hln` and `ar_pln` to compute offsets.

## ARP Table Initialization

ARP registers with the neighboring subsystem via `arp_tbl` (a `struct neigh_table`):

- **Constructor**: `arp_constructor()` -- called when a new `neighbour` is created. Sets up `neigh_ops` based on device type (Ethernet gets `arp_generic_ops`, loopback gets `arp_direct_ops`).
- **Solicitation**: `arp_solicit()` -- sends ARP requests. Called by the NUD state machine when resolution is needed.
- **Error report**: `arp_error_report()` -- reports resolution failure to upper layers (sends `EHOSTUNREACH`).

The `arp_tbl` has hash function `arp_hash()` which hashes the IPv4 address and device index.

## Key Functions

### Transmission

| Function | Role |
|----------|------|
| `arp_solicit()` | Send ARP request for unresolved neighbor. Selects source address via `arp_announce` policy. |
| `arp_send()` | Low-level: allocates `sk_buff`, fills ARP header, calls `dev_queue_xmit()` |
| `arp_create()` | Allocate and populate an ARP packet `sk_buff` |

### Reception

| Function | Role |
|----------|------|
| `arp_rcv()` | Entry point from `netif_receive_skb()`. Validates packet, applies netfilter `NF_ARP_IN` hook, calls `arp_process()` |
| `arp_process()` | Main processing: handles requests (sends reply, updates cache) and replies (updates neighbor entry). Core logic for gratuitous ARP, proxy ARP, and duplicate address detection. |

### `arp_process()` Logic

For incoming ARP packets, `arp_process()` follows this decision flow:

1. **Validate**: Check packet fields, reject packets from/to loopback or multicast source IPs.
2. **Lookup neighbor**: Call `neigh_lookup()` for the sender IP.
3. **If ARP reply**: Update the existing neighbor entry via `neigh_update()` with the sender's hardware address.
4. **If ARP request**:
   a. Check if target IP is a local address (`ip_route_input()` returns RTN_LOCAL).
   b. If local: send ARP reply via `arp_send()`, update sender's neighbor entry.
   c. If not local, check proxy ARP (see below).
   d. If proxy enabled and route exists: either reply immediately or queue for delayed proxy response.
5. **Update existing entries**: Even if we don't respond, we update stale entries with the sender's address if we already have an entry (the "update on any valid ARP" optimization).

## Gratuitous ARP

A gratuitous ARP is a request where the sender and target IP are the same. Sent when:

- An interface is configured with a new IP address (`arp_announce` on address addition)
- Used for **duplicate address detection**: if a reply comes back, another host has the same IP
- Used to **update other hosts' caches**: all hosts receiving the gratuitous ARP update their existing entries for that IP

The kernel sends gratuitous ARPs from `arp_netdev_event()` (on `NETDEV_CHANGEADDR`) and when an address is added via `inetdev_event()`.

## Proxy ARP

Proxy ARP allows a host to answer ARP requests on behalf of other hosts, typically to bridge subnets transparently.

**Configuration**: Enabled per-interface via `/proc/sys/net/ipv4/conf/<interface>/proxy_arp`.

**Mechanism**:
1. `arp_process()` checks if proxy ARP is enabled for the receiving interface.
2. If the target IP is routable through a different interface (checked via `fib_lookup()`), the host can proxy.
3. Proxy entries can be explicitly added: `ip neigh add proxy <ip> dev <iface>`, stored in `phash_buckets` of the `neigh_table`.
4. `pneigh_lookup()` checks for explicit proxy entries.
5. Proxy responses are delayed randomly (up to `proxy_delay`) to avoid collisions when multiple proxies exist, queued via `proxy_timer`.

**arp_filter**: When `/proc/sys/net/ipv4/conf/*/arp_filter` is enabled, the kernel only responds to ARP requests if the target IP would be routed through the receiving interface. This prevents responding on the "wrong" interface on multi-homed hosts.

**arp_announce levels**:
- 0: Use any local address (default)
- 1: Prefer address on the outgoing interface
- 2: Use only the best local address (most restrictive)

**arp_ignore levels**:
- 0: Reply to any ARP request for any local address (default)
- 1: Reply only if target IP is configured on the receiving interface

## ARPD (User-Space ARP Daemon)

When compiled with `CONFIG_ARPD`, the kernel can delegate ARP table management to a user-space daemon:

- Communication via **netlink socket** (`NETLINK_ARPD`)
- Kernel sends `ARPD_LOOKUP` when it needs to resolve an address
- Daemon responds with `ARPD_UPDATE` containing the mapping
- Controlled by `app_solicit` parameter (number of solicitations sent to daemon before falling back to kernel ARP)
- Useful for large L2 networks or custom resolution logic

## RARP (Reverse ARP)

Reverse ARP (mapping hardware address to IP address) is largely obsolete, replaced by BOOTP and DHCP. The kernel retains minimal RARP support:

- `CONFIG_RARP`: Compile-time option
- Used only for diskless boot scenarios
- `rarp_rcv()`: Processes incoming RARP replies during boot

## External Events

ARP responds to these kernel notifications:

| Event | Handler | Action |
|-------|---------|--------|
| `NETDEV_CHANGEADDR` | `arp_netdev_event()` | Send gratuitous ARP with new hardware address |
| `NETDEV_DOWN` | (neighboring core) | Flush all neighbor entries for the device |
| IP address added | `inetdev_event()` | Send gratuitous ARP for the new address |

## Administration

```bash
# View ARP cache
ip neigh show

# Add static entry
ip neigh add 192.168.1.1 lladdr 00:11:22:33:44:55 dev eth0 nud permanent

# Delete entry
ip neigh del 192.168.1.1 dev eth0

# Flush cache
ip neigh flush dev eth0

# Add proxy ARP entry
ip neigh add proxy 192.168.1.100 dev eth0
```

The legacy `arp` command uses ioctl (`SIOCSARP`, `SIOCGARP`, `SIOCDARP`), while `ip neigh` uses netlink, which is preferred.

## See also

- [neighboring-subsystem](neighboring-subsystem.md)
- [routing-subsystem](routing-subsystem.md)
- [concept-network-packet-flow](../concepts/concept-network-packet-flow.md)
- [src-understanding-linux-network-internals](../sources/src-understanding-linux-network-internals.md)
