---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [understanding-linux-network-internals]
tags: [networking, bridging, stp, spanning-tree, layer2, forwarding-database]
---

# Bridging

The Linux bridge is a software implementation of an IEEE 802.1D Ethernet bridge. It operates at Layer 2, forwarding frames between ports based on MAC addresses. The bridge appears as a virtual `net_device` with physical interfaces attached as slave ports.

## Key Data Structures

### net_bridge

The main bridge structure, extending `net_device`:

- `port_list` — linked list of `net_bridge_port` entries (slave interfaces)
- `hash[BR_HASH_SIZE]` — forwarding database hash table
- `bridge_id` — this bridge's ID (2-byte priority + 6-byte MAC)
- `designated_root` — root bridge ID as determined by STP
- `root_path_cost` — cumulative cost to reach the root
- `max_age`, `hello_time`, `forward_delay` — STP timer parameters
- `topology_change`, `topology_change_detected` — TC flags
- `hello_timer`, `tcn_timer`, `topology_change_timer` — STP timers
- `ageing_time` — forwarding database entry lifetime (default 300s)
- `stp_enabled` — whether STP is active

### net_bridge_port

A slave interface attached to a bridge:

- `dev` — pointer to the underlying `net_device`
- `br` — back-pointer to the `net_bridge`
- `port_id` — port number within the bridge
- `state` — STP port state (disabled/blocking/listening/learning/forwarding)
- `priority` — per-port STP priority
- `path_cost` — STP path cost for this port
- `designated_root`, `designated_bridge`, `designated_port` — STP designated info
- `message_age_timer`, `forward_delay_timer`, `hold_timer` — per-port STP timers

### net_bridge_fdb_entry (Forwarding Database Entry)

- `addr` — MAC address (the lookup key)
- `dst` — pointer to the `net_bridge_port` where this MAC was learned
- `ageing_timer` — jiffies timestamp of last activity
- `is_local` — this is the bridge's own MAC (never ages)
- `is_static` — statically configured entry (never ages)

## Forwarding Database

The forwarding database (fdb) maps MAC addresses to bridge ports. It uses a hash table indexed by MAC address.

### Learning

When a frame arrives on a port in LEARNING or FORWARDING state, `br_fdb_update()` creates or refreshes an fdb entry for the source MAC address, recording which port it was seen on.

### Aging

A periodic timer (`gc_timer`) runs `br_fdb_cleanup()`, which walks the fdb and removes learned entries whose `ageing_timer` has expired (older than `ageing_time`, default 300 seconds). Local and static entries never age.

During a topology change, the aging time is temporarily shortened to `forward_delay` (typically 15 seconds) to speed up convergence.

### Lookup and Forwarding

`br_fdb_get()` looks up a destination MAC:
- **Found, unicast**: Forward to the specific port
- **Not found**: Flood to all ports except the source
- **Local entry**: Deliver up to the bridge's own network stack

## Spanning Tree Protocol (STP)

Linux implements IEEE 802.1D STP to prevent L2 loops in redundant topologies.

### Port States

| State | Learn MACs | Forward frames | Receive BPDUs |
|-------|-----------|---------------|---------------|
| Disabled | No | No | No |
| Blocking | No | No | Yes |
| Listening | No | No | Yes |
| Learning | Yes | No | Yes |
| Forwarding | Yes | Yes | Yes |

Transitions: Disabled -> Blocking -> Listening -> Learning -> Forwarding. The listening-to-learning and learning-to-forwarding transitions each take `forward_delay` seconds (default 15s).

### Bridge and Port IDs

- Bridge ID = 2-byte priority (default 0x8000) + 6-byte MAC address
- Port ID = 1-byte priority + 1-byte port number
- The bridge with the lowest Bridge ID becomes the root bridge

### Bridge Protocol Data Units (BPDUs)

Two types:
- **Configuration BPDU**: Contains root ID, root path cost, sender's bridge/port ID, timers. Sent by designated ports.
- **Topology Change Notification (TCN) BPDU**: Sent toward the root when a topology change is detected.

### Key STP Functions

- `br_received_config_bpdu()` — processes incoming Configuration BPDUs; updates root/designated info, triggers state changes
- `br_received_tcn_bpdu()` — processes TCN BPDUs; sets TC flag and relays toward root
- `br_config_bpdu_generation()` — sends Configuration BPDUs on all designated ports (on hello timer expiry)
- `br_topology_change_detection()` — detects TC events (port entering forwarding or leaving forwarding)
- `br_become_root_bridge()` — called when this bridge determines it is the root

### Topology Change Handling

When a port transitions to forwarding or a forwarding port goes down:
1. Non-root bridges send TCN BPDUs toward the root
2. The root sets the TC flag in its Configuration BPDUs for `max_age + forward_delay` seconds
3. All bridges receiving TC-flagged BPDUs shorten their fdb aging time
4. Stale entries flush out quickly, enabling convergence

## Frame Processing

### Ingress Path

`br_handle_frame()` is called from `netif_receive_skb()` for interfaces enslaved to a bridge:

1. If the frame is a BPDU (destination = bridge group address 01:80:C2:00:00:00), process for STP
2. If the port is in FORWARDING state:
   - Learn the source MAC (`br_fdb_update()`)
   - Look up the destination MAC (`br_fdb_get()`)
   - If the destination is local (the bridge itself), deliver to the network stack via `br_pass_frame_up()`
   - If the destination is known on another port, forward via `br_forward()`
   - If the destination is unknown or multicast/broadcast, flood via `br_flood()`
3. If the port is in LEARNING state, learn but don't forward

### Egress Path

`br_dev_xmit()` handles transmission from the bridge device (packets originating from the bridge's own IP stack). Looks up the destination MAC and forwards or floods as above.

## User-Space Configuration

The `brctl` utility manages bridges via ioctls:

- `brctl addbr <name>` / `brctl delbr <name>` — create/delete a bridge
- `brctl addif <bridge> <if>` / `brctl delif <bridge> <if>` — add/remove ports
- `brctl stp <bridge> on|off` — enable/disable STP
- `brctl showmacs <bridge>` — display forwarding database
- `brctl show` — list bridges and ports

Modern alternative: `ip link add <name> type bridge`, `ip link set <if> master <bridge>`

Sysfs entries: `/sys/class/net/<bridge>/bridge/` (per-bridge), `/sys/class/net/<bridge>/brif/<port>/` (per-port)

## See also

- [frame-reception](frame-reception.md)
- [net-device](net-device.md)
- [concept-network-packet-flow](../concepts/concept-network-packet-flow.md)
- [neighboring-subsystem](neighboring-subsystem.md)
