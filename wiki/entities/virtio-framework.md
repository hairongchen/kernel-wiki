---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [qemu-kvm-source-code-and-application]
tags: [virtio, vring, virtqueue, paravirtualization, virtio-net, virtio-blk]
---

# Virtio Virtualization Framework

Virtio is a standardized paravirtualized I/O interface for virtual machines. Rather than emulating real hardware register-by-register, virtio defines an explicit guest-host communication protocol, allowing the guest kernel to cooperate with the hypervisor for dramatically better I/O performance. The framework is used throughout [qemu-kvm-overview](qemu-kvm-overview.md) for network, block, and other device classes.

## Architecture

Virtio is organized into three layers:

1. **Frontend driver** (guest kernel) -- The virtio device driver running inside the guest OS. It speaks the virtio protocol and places I/O requests into shared-memory queues.
2. **Backend device** (QEMU userspace) -- The device model in QEMU that consumes requests from the shared queues, performs actual I/O on the host, and posts completions back.
3. **Transport layer** -- The mechanism by which the guest discovers virtio devices and maps the shared memory regions. The two main transports are PCI (virtio-pci) and MMIO (virtio-mmio). PCI is dominant on x86; MMIO is common on ARM and embedded platforms.

This separation means the same frontend driver works regardless of the transport, and the same backend works regardless of the guest OS.

## VRing Data Structures

The core shared-memory data structure is the **VRing** (virtio ring), which consists of three regions laid out in guest-physical memory.

### VRingDesc (Descriptor Table)

Each descriptor describes one buffer:

| Field   | Description |
|---------|-------------|
| `addr`  | Guest physical address (GPA) of the buffer |
| `len`   | Length of the buffer in bytes |
| `flags` | Bit flags: `VRING_DESC_F_NEXT` (chained descriptor follows), `VRING_DESC_F_WRITE` (buffer is device-writable, i.e., for host-to-guest data), `VRING_DESC_F_INDIRECT` (buffer contains a table of indirect descriptors) |
| `next`  | Index of the next descriptor in the chain (valid only when NEXT flag is set) |

Multiple descriptors are chained via `next` indices to describe a single I/O request that spans non-contiguous buffers. The indirect flag allows a single descriptor to point to a separately-allocated table of descriptors, expanding capacity without enlarging the main ring.

### VRingAvail (Available Ring)

Written by the **guest**, read by the **host**. The guest uses this ring to offer descriptor chains to the device:

| Field    | Description |
|----------|-------------|
| `flags`  | `VRING_AVAIL_F_NO_INTERRUPT` -- guest asks host not to send interrupts after consuming entries |
| `idx`    | Monotonically increasing counter of total entries added (wraps at 2^16) |
| `ring[]` | Array of descriptor head indices, each identifying the first descriptor of a chain |

### VRingUsed (Used Ring)

Written by the **host**, read by the **guest**. The host uses this ring to return completed descriptor chains:

| Field    | Description |
|----------|-------------|
| `flags`  | `VRING_USED_F_NO_NOTIFY` -- host asks guest not to kick after adding new available entries |
| `idx`    | Monotonically increasing counter of total entries added |
| `ring[]` | Array of `VRingUsedElem` structs |

Each `VRingUsedElem` contains `id` (the head descriptor index being returned) and `len` (total bytes written into the descriptor chain by the device).

### VRing

The top-level `VRing` struct combines: `num` (queue size, always a power of two), plus pointers to the `desc`, `avail`, and `used` arrays.

## VirtQueue in QEMU

QEMU's `VirtQueue` structure wraps a VRing with additional bookkeeping for the backend:

| Field                | Purpose |
|----------------------|---------|
| `vring`              | The underlying VRing mapped from guest memory |
| `last_avail_idx`     | Host-side shadow of how far it has consumed the available ring |
| `shadow_avail_idx`   | Cached copy of `vring.avail->idx` to reduce guest memory reads |
| `used_idx`           | Host-side shadow of how far it has written the used ring |
| `signalled_used`     | Last `used_idx` value at which an interrupt was sent (for interrupt coalescing) |
| `notification`       | Boolean flag controlling whether the guest should notify the host on new entries |
| `handle_output`      | Callback function invoked when the guest kicks this queue |
| `guest_notifier`     | `EventNotifier` (eventfd) for injecting interrupts into the guest via [kvm-interrupt-virtualization](kvm-interrupt-virtualization.md) |
| `host_notifier`      | `EventNotifier` (eventfd) for receiving kicks from the guest (used with ioeventfd) |

## VirtIODevice Base Class

All virtio device models in QEMU inherit from `VirtIODevice`:

**Key fields:**

- `name` -- human-readable device name
- `device_id` -- virtio device type identifier (e.g., 1 = net, 2 = blk)
- `status` -- device status register (ACKNOWLEDGE, DRIVER, DRIVER_OK, FEATURES_OK, FAILED)
- `isr` -- interrupt status register
- `host_features` / `guest_features` -- feature bit negotiation between host and guest
- `config` -- device-specific configuration space
- `vq[]` -- array of VirtQueues belonging to this device

**VirtioDeviceClass methods:**

| Method                 | Role |
|------------------------|------|
| `realize()`            | Device initialization -- allocates queues, registers features |
| `get_config()` / `set_config()` | Read/write device-specific configuration |
| `get_features()` / `set_features()` | Feature negotiation: host advertises, guest accepts a subset |
| `reset()`              | Restore device to initial state |

## Core Virtqueue Operations

### Creating a queue

`virtio_add_queue(vdev, queue_size, handle_output)` -- Allocates a VirtQueue of the given size and registers the callback invoked when the guest writes to the notification register.

### Consuming requests (host side)

`virtqueue_pop(vq, sz)` -- Dequeues the next available descriptor chain:

1. Reads the head index from `avail->ring[last_avail_idx % num]`.
2. Walks the descriptor chain following `next` indices.
3. Separates buffers into device-readable (out_sg, guest-to-host data such as a write payload) and device-writable (in_sg, host-to-guest data such as a read result or status byte).
4. Returns a `VirtQueueElement` containing `index`, `in_num`, `out_num`, `in_sg[]`, and `out_sg[]`.
5. Advances `last_avail_idx`.

### Posting completions

- `virtqueue_push(vq, elem, len)` -- Convenience wrapper that calls `virtqueue_fill()` then `virtqueue_flush()` for a single element.
- `virtqueue_fill(vq, elem, len, idx)` -- Writes one entry into the used ring at the given offset without updating `used->idx`. Allows batching multiple completions.
- `virtqueue_flush(vq, count)` -- Advances `used->idx` by `count`, making the batch visible to the guest. A write memory barrier ensures the used ring entries are visible before the index update.

### Notifying the guest

`virtio_notify(vdev, vq)` -- Injects an interrupt into the guest (via the PCI transport's MSI-X or legacy IRQ path) to signal that new used ring entries are available. Interrupt coalescing logic compares `signalled_used` against the current `used_idx` to suppress redundant interrupts.

## PCI Transport (VirtIOPCIProxy)

`VirtIOPCIProxy` wraps any `VirtIODevice` as a PCI device, making it discoverable through the guest's standard PCI enumeration.

### Modern virtio 1.0

The modern transport maps device registers through PCI capabilities, each in its own BAR region:

| Capability     | Purpose |
|----------------|---------|
| Common cfg     | Device status, feature negotiation, queue configuration |
| Notify         | Per-queue doorbell registers (guest writes here to kick) |
| ISR            | Interrupt status readback |
| Device cfg     | Device-specific configuration space |

### Legacy transport

A single I/O BAR with a fixed register layout at well-known offsets. Still supported for backward compatibility with older guests.

### Initialization

`virtio_pci_realize()` sets up the PCI BARs and registers the capability structures. When the guest kicks a queue by writing to the notification BAR, the kernel's KVM ioeventfd mechanism intercepts the write in-kernel and signals the host_notifier eventfd, waking QEMU's `handle_output` callback without a full VM exit to userspace. This is critical for performance.

## Virtio-net

`VirtIONet` is the paravirtualized network device, the most widely used virtio device type.

### Structure

- Array of `VirtIONetQueue` structs, each containing an `rx_vq` (receive) and `tx_vq` (transmit) virtqueue -- one pair per queue for multiqueue support.
- `ctrl_vq` -- control virtqueue for setting MAC address, VLAN filters, multiqueue parameters.
- `NICState` -- QEMU's generic NIC abstraction connecting to a network backend.

### Transmit path

1. Guest places packet buffers in `tx_vq` and kicks.
2. `virtio_net_flush_tx()` is called, which loops calling `virtqueue_pop()` to dequeue packets.
3. Each packet is sent to the network backend via `qemu_sendv_packet_async()`, typically reaching a TAP device.
4. On completion, `virtqueue_push()` returns the descriptor and `virtio_notify()` signals the guest.

### Receive path

1. The TAP backend calls `virtio_net_receive()` when a packet arrives.
2. The function dequeues pre-posted receive buffers from `rx_vq` via `virtqueue_pop()`.
3. Packet data is copied into the guest buffers via `virtqueue_fill()`.
4. `virtqueue_flush()` and `virtio_notify()` deliver the completion.

### Packet header

Every packet is prefixed with a `virtio_net_hdr` containing:

- `flags` -- checksum offload request (`VIRTIO_NET_HDR_F_NEEDS_CSUM`)
- `gso_type` -- segmentation offload type (TCPv4, TCPv6, UDP, NONE)
- `hdr_len`, `gso_size` -- GSO parameters
- `csum_start`, `csum_offset` -- partial checksum offload location

### Mergeable RX buffers

When `VIRTIO_NET_F_MRG_RXBUF` is negotiated, a single large incoming packet can span multiple receive buffers. The header includes a `num_buffers` field indicating how many descriptors were consumed. This avoids requiring the guest to pre-post buffers large enough for the maximum MTU.

### TAP backend

`TAPState` manages the host TAP file descriptor. `tap_send()` reads from the fd and forwards packets into the virtio-net receive path. `tap_receive_iov()` writes packet data from the transmit path into the TAP fd. For improved performance, the [vhost](vhost.md) kernel module can handle the data plane entirely in the host kernel, bypassing QEMU userspace.

## Virtio-blk

`VirtIOBlock` is the paravirtualized block (disk) device.

### Structure

- `BlockBackend` -- QEMU's generic block layer abstraction connecting to an image file or host device.
- `VirtIOBlockReq` -- per-request state containing the `VirtQueueElement`, the request header, and a `QEMUIOVector` for scatter-gather I/O.

### Request format

Each block request begins with a `virtio_blk_outhdr` (device-readable) containing:

| Field    | Description |
|----------|-------------|
| `type`   | `VIRTIO_BLK_T_IN` (read), `VIRTIO_BLK_T_OUT` (write), `VIRTIO_BLK_T_FLUSH`, `VIRTIO_BLK_T_GET_ID` |
| `ioprio` | I/O priority (reserved) |
| `sector` | Starting sector for the operation |

The last byte of the request is a device-writable `virtio_blk_inhdr` containing a `status` field (0 = OK, 1 = IOERR, 2 = UNSUPPORTED).

### Request processing

- **Read** (`VIRTIO_BLK_T_IN`): Calls `blk_aio_preadv()` to asynchronously read from the block backend into the guest's writable scatter-gather buffers.
- **Write** (`VIRTIO_BLK_T_OUT`): Calls `blk_aio_pwritev()` to asynchronously write the guest's readable scatter-gather buffers to disk.
- **Flush** (`VIRTIO_BLK_T_FLUSH`): Issues a cache flush to the block backend.
- **Get ID** (`VIRTIO_BLK_T_GET_ID`): Returns the disk serial number.

### Completion

`virtio_blk_rw_complete()` is the async callback invoked when the block backend finishes an operation. It sets the status byte, calls `virtqueue_push()`, and triggers `virtio_notify()`.

### IOThread data plane

For high-performance configurations, virtio-blk supports an **IOThread data plane** model. A dedicated thread handles the virtqueue polling and block I/O submission, offloading work from QEMU's main event loop. This reduces lock contention and improves IOPS, especially for NVMe-backed storage.

## Other Virtio Devices

| Device | Purpose |
|--------|---------|
| **virtio-balloon** | Dynamic guest memory management. The host requests the guest to inflate (return pages) or deflate (reclaim pages) a memory balloon, enabling overcommit and live memory balancing. Important during [vm-live-migration](vm-live-migration.md). |
| **virtio-serial** | Multiplex character streams between host and guest. Primary use case is the QEMU guest agent (`qemu-ga`), which accepts commands (freeze filesystems, sync time) over a virtio-serial channel. |
| **virtio-rng** | Provides entropy from the host to the guest. The guest reads from a virtqueue and receives random bytes sourced from the host's `/dev/urandom` or a hardware RNG. Addresses the entropy starvation problem in VMs. |
| **virtio-gpu** | Paravirtualized GPU supporting 2D scanout and optional 3D acceleration via virglrenderer. Provides efficient display rendering without full GPU emulation. |

## See also

- [qemu-kvm-overview](qemu-kvm-overview.md)
- [vhost](vhost.md)
- [kvm-interrupt-virtualization](kvm-interrupt-virtualization.md)
- [vm-live-migration](vm-live-migration.md)
