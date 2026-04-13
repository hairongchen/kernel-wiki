---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [professional-linux-kernel-architecture]
tags: [networking, sockets, bsd-sockets, inet, proto-ops]
---

# Socket Layer

The socket layer provides the BSD socket API to user space and bridges application-level I/O to the kernel's transport protocols. It is organized around a **two-level structure**: `struct socket` exposes the VFS/BSD interface that user-space programs interact with, while `struct sock` holds the network-layer protocol state. This separation lets the upper half remain protocol-independent while the lower half implements protocol-specific behavior.

Defined primarily in `net/socket.c`, `include/linux/net.h`, `include/net/sock.h`, and `net/ipv4/af_inet.c`.

## struct socket — BSD-Level Representation

`struct socket` is the VFS-facing half of a socket. It is what the kernel associates with a file descriptor and what generic socket syscalls operate on.

### Key Fields

| Field | Description |
|-------|-------------|
| `state` | Connection state: `SS_UNCONNECTED`, `SS_CONNECTING`, `SS_CONNECTED`, `SS_DISCONNECTING` |
| `type` | Socket type: `SOCK_STREAM`, `SOCK_DGRAM`, `SOCK_RAW`, etc. |
| `flags` | Per-socket flags (e.g., `SOCK_ASYNC_NOSPACE`) |
| `ops` | Pointer to `struct proto_ops` — the address-family-specific operation table |
| `sk` | Pointer to `struct sock` — the underlying network-layer socket |
| `file` | Back-pointer to the VFS `struct file` associated with this socket |

### Allocation

`sock_alloc()` creates a socket by allocating a **socket inode** on the `sockfs` pseudo-filesystem. The `sockfs` filesystem is mounted internally during boot; it never appears in user-visible mount tables. The `struct socket` is embedded within `struct socket_alloc`, which also contains the VFS `struct inode`, so a single allocation produces both the socket and its inode.

## struct sock — Network-Layer Representation

`struct sock` (commonly referred to as "sk") is the protocol-facing half. It holds per-connection state that transport protocols like TCP and UDP need: buffer queues, protocol state machines, timers, and memory accounting.

### Key Fields

| Field | Description |
|-------|-------------|
| `sk_state` | Protocol-level state (e.g., `TCP_ESTABLISHED`, `TCP_CLOSE`) |
| `sk_rcvbuf` | Maximum receive buffer size (bytes) |
| `sk_sndbuf` | Maximum send buffer size (bytes) |
| `sk_receive_queue` | Queue of incoming [sk-buff](sk-buff.md) packets awaiting consumption |
| `sk_write_queue` | Queue of outgoing sk_buffs awaiting transmission |
| `sk_prot` | Pointer to `struct proto` — transport protocol operations |
| `sk_sleep` | Wait queue head for processes sleeping on this socket |
| `sk_refcnt` | Reference count for safe destruction |

### Protocol-Specific Extensions

The `struct sock` is extended via C struct embedding for each protocol family:

- **`struct inet_sock`** — extends `struct sock` with IPv4-specific fields:
  - `inet_saddr` / `inet_daddr` — local and remote IPv4 addresses
  - `inet_sport` / `inet_dport` — local and remote port numbers
  - `inet_opt` — pointer to IP options (`struct ip_options`)

- **`struct tcp_sock`** — extends `struct inet_sock` with TCP state: sequence numbers, congestion control variables, retransmission timers, SACK scoreboard, window scaling parameters.

- **`struct udp_sock`** — extends `struct inet_sock` with UDP-specific fields such as the pending frames list and encapsulation receive hook.

Because C struct embedding places the parent at offset zero, a pointer to `struct tcp_sock` can be safely cast to `struct inet_sock` or `struct sock`.

## struct proto_ops — VFS-Facing Operations

`struct proto_ops` defines the operations that the generic socket code calls. Each address family provides its own instance. These methods are the bridge between the protocol-independent `sys_bind()`, `sys_connect()`, etc. and the address-family-specific implementation.

### Methods

| Method | Purpose |
|--------|---------|
| `bind` | Bind socket to a local address |
| `connect` | Initiate a connection (or set peer for datagrams) |
| `accept` | Accept an incoming connection |
| `listen` | Mark socket as passive (listening) |
| `sendmsg` | Send a message |
| `recvmsg` | Receive a message |
| `poll` | Check readiness for I/O multiplexing |
| `ioctl` | Socket-specific ioctl commands |
| `shutdown` | Shut down part or all of a connection |
| `setsockopt` / `getsockopt` | Get/set socket options |
| `release` | Final cleanup when the socket is closed |

### AF_INET Instances

| Instance | Protocol | Type |
|----------|----------|------|
| `inet_stream_ops` | TCP | `SOCK_STREAM` |
| `inet_dgram_ops` | UDP | `SOCK_DGRAM` |
| `inet_sockraw_ops` | Raw IP | `SOCK_RAW` |

## struct proto — Transport Protocol Operations

`struct proto` defines the operations that the transport protocol implements. Where `proto_ops` faces the VFS/socket layer, `proto` faces the network stack. Each transport protocol provides a single global instance.

### Methods

| Method | Purpose |
|--------|---------|
| `connect` | Protocol-level connection setup |
| `disconnect` | Tear down connection state |
| `accept` | Accept on the protocol side |
| `sendmsg` | Transmit data (e.g., build segments) |
| `recvmsg` | Deliver data to user space |
| `bind` | Protocol-level bind logic |
| `hash` / `unhash` | Insert/remove socket from protocol hash table |
| `get_port` | Allocate or verify a port number |
| `init` | Per-socket protocol initialization |
| `close` | Protocol-level close and cleanup |

### Key Instances

- **`tcp_prot`** — TCP transport operations (defined in `net/ipv4/tcp_ipv4.c`)
- **`udp_prot`** — UDP transport operations (defined in `net/ipv4/udp.c`)

## Socket Creation Flow

When user space calls `socket(AF_INET, SOCK_STREAM, 0)`, the kernel executes `sys_socket(family, type, protocol)`:

1. **`sock_create()`** → calls **`__sock_create()`**
2. **`__sock_create()`** looks up the `struct net_proto_family` registered for the requested address family. For `AF_INET`, this is `inet_family_ops` (registered by `inet_init()` at boot).
3. Calls **`family->create()`**, which for AF_INET dispatches to **`inet_create()`**.
4. **`inet_create()`** searches the `struct inet_protosw` table — an array indexed by socket type and protocol number — to find the matching `struct proto` and `struct proto_ops`. For `(SOCK_STREAM, IPPROTO_TCP)` it finds `tcp_prot` and `inet_stream_ops`.
5. Allocates a `struct sock` (actually `struct tcp_sock` for TCP) via `sk_alloc()`, initializes `struct inet_sock` fields (addresses zeroed, ports unset).
6. Calls `sock->ops->init()` if defined — for TCP this runs `tcp_v4_init_sock()`, which sets up initial congestion control, timers, and sequence number state.
7. **`sock_map_fd()`** creates a VFS `struct file` backed by the socket and installs it in the process's file descriptor table. Returns the new file descriptor to user space.

## Socket Syscall Dispatch

All socket-related syscalls eventually resolve a file descriptor to a `struct socket` and call through `socket->ops` (the `proto_ops` table).

- **x86-32**: Socket calls are multiplexed through a single entry point, `sys_socketcall(call, args)`. The `call` argument selects the operation (SYS_BIND, SYS_CONNECT, etc.) and `args` is a pointer to the parameters. This design dates back to early Linux when syscall numbers were scarce.

- **x86-64**: Each socket operation has its own syscall number (`sys_bind`, `sys_connect`, `sys_listen`, etc.), dispatched directly through the syscall table.

In both cases, the kernel resolves the file descriptor to a `struct socket` via `sockfd_lookup()`, which walks `current->files` to find the `struct file`, then extracts the socket from the file's private data.

## Data Flow

### Send Path

```
sys_sendto() / sys_sendmsg()
  → sock_sendmsg()
    → security_socket_sendmsg()          [LSM hook]
    → socket->ops->sendmsg()             [proto_ops, e.g., inet_sendmsg]
      → sk->sk_prot->sendmsg()           [struct proto, e.g., tcp_sendmsg]
        → build segments, queue on sk_write_queue
        → tcp_push() → ip_queue_xmit()   [hand off to IP layer]
```

For TCP, `tcp_sendmsg()` copies user data into sk_buffs on the write queue, coalescing into MSS-sized segments where possible. It then calls `tcp_push()` to trigger actual transmission through the [ipv4-subsystem](ipv4-subsystem.md).

### Receive Path

```
sys_recvfrom() / sys_recvmsg()
  → sock_recvmsg()
    → security_socket_recvmsg()          [LSM hook]
    → socket->ops->recvmsg()             [proto_ops, e.g., inet_recvmsg]
      → sk->sk_prot->recvmsg()           [struct proto, e.g., tcp_recvmsg]
        → copy from sk_receive_queue to user buffer
```

For TCP, `tcp_recvmsg()` walks the `sk_receive_queue`, copying contiguous in-sequence data to the user buffer. It manages TCP sequence numbers, checks for urgent data, and may block (for blocking sockets) if insufficient data is available. As data is consumed, the receive window opens and ACKs are sent to the peer.

## Socket Buffers and Flow Control

The socket layer enforces per-socket memory limits to prevent a single connection from consuming unbounded kernel memory.

- **Receive side**: `sk_rcvbuf` sets the maximum memory for queued incoming packets. `sock_queue_rcv_skb()` checks whether accepting a new [sk-buff](sk-buff.md) would exceed this limit; if so, the packet is dropped. The value is tunable via `SO_RCVBUF` / `sysctl net.core.rmem_default`.

- **Send side**: `sk_sndbuf` limits the memory consumed by the write queue. When the write queue exceeds this limit, `sendmsg()` blocks (for blocking sockets) or returns `EAGAIN` (for non-blocking sockets) until the transport protocol frees sk_buffs after successful transmission and acknowledgment.

- **Autotuning**: The kernel can dynamically adjust buffer sizes within `sysctl net.ipv4.tcp_rmem` / `tcp_wmem` min-default-max ranges based on observed traffic patterns and available memory.

## See also

- [sk-buff](sk-buff.md) — Socket buffer structure used for all packet data
- [ipv4-subsystem](ipv4-subsystem.md) — IPv4 protocol layer called from socket send path
- [ipc](ipc.md) — Inter-process communication including UNIX domain sockets
- [concept-network-packet-flow](../concepts/concept-network-packet-flow.md) — End-to-end packet traversal through the networking stack
