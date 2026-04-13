---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [qemu-kvm-source-code-and-application]
tags: [vhost, vhost-net, virtio, ioeventfd, irqfd, data-plane]
---

# Vhost: Kernel-Based Virtio Backend

Vhost is a Linux kernel infrastructure that accelerates virtio device emulation by
moving the **data plane** from QEMU userspace into the host kernel. QEMU retains
the **control plane** — configuration, feature negotiation, and
[vm-live-migration](vm-live-migration.md) support — while the kernel handles the hot path of
packet processing directly. This eliminates costly context switches between the
guest, KVM, and QEMU on every I/O operation.

Two backend types exist:

- **vhost-net** — a kernel module (`drivers/vhost/net.c`) exposed as the misc
  device `/dev/vhost-net`. This is the focus of this page.
- **vhost-user** — a separate userspace process (e.g., DPDK) communicating with
  QEMU over a Unix domain socket. Same protocol, but the data plane runs in a
  different userspace process rather than the kernel.

## Architecture Overview

In the standard [virtio-framework](virtio-framework.md) model, every guest I/O triggers a
VM exit into KVM, which returns to QEMU userspace, which processes the request
and re-enters the guest. With vhost-net, the flow becomes:

1. Guest writes to the virtio notification register.
2. KVM intercepts the write via **ioeventfd** — no exit to QEMU.
3. The ioeventfd signals an eventfd monitored by the vhost kernel thread.
4. The vhost worker thread processes packets directly from the virtqueue,
   reading guest memory through QEMU's page tables.
5. After filling the used ring, vhost signals an **irqfd** that injects an
   interrupt into the guest — again bypassing QEMU entirely.

Both the notification path (guest-to-host) and the completion path
(host-to-guest) completely bypass QEMU userspace.

## QEMU-Side Data Structures

### vhost_dev

The central QEMU-side structure representing a vhost device:

- `vdev` — pointer to the associated VirtIODevice.
- `memory_listener` — registered with QEMU's memory subsystem to track guest
  physical memory changes and relay them to the kernel via
  `VHOST_SET_MEM_TABLE`.
- `vqs` — array of `vhost_virtqueue` structures, one per virtqueue.
- `vhost_ops` / `VhostOps` — function pointer table dispatching operations to
  the appropriate backend (kernel or user).

### vhost_net

Wraps `vhost_dev` with networking-specific state:

- Two `vhost_virtqueue` entries (TX and RX).
- `backend` — the file descriptor for the TAP device.
- `nc` — a `NetClientState` linking to QEMU's networking layer.

### VhostOps (kernel_ops)

For the kernel backend, every function in the `VhostOps` table calls
`vhost_kernel_call()`, which reduces to:

```c
ioctl(fd, request, arg);
```

where `fd` is the file descriptor obtained from opening `/dev/vhost-net`.

### TAPState

QEMU's representation of a TAP network backend. Contains a `vhost_net` member
that is allocated when vhost acceleration is enabled for the TAP device.

## QEMU-Side Initialization

### vhost_dev_init()

Called during device realization. The sequence:

1. **Set backend type** — kernel or user.
2. **vhost_backend_init()** — opens `/dev/vhost-net`, obtaining a file
   descriptor.
3. **VHOST_SET_OWNER ioctl** — tells the kernel module to create a dedicated
   worker thread for this device instance.
4. **vhost_get_features()** — queries the kernel module for supported virtio
   feature bits.
5. **vhost_virtqueue_init()** — called for each queue:
   - Sets up a `masked_notifier` eventfd.
   - Issues **VHOST_SET_VRING_CALL** to register the eventfd used for
     host-to-guest notification.
6. **Registers memory_listener** — ensures that any future guest memory layout
   changes (hotplug, balloon, etc.) are forwarded to the kernel module for
   dirty page tracking.

### Startup Sequence

When the guest driver activates the virtio-net device:

1. `virtio_net_vhost_status()` detects the device is ready (VIRTIO_CONFIG_S_DRIVER_OK).
2. Calls `vhost_net_start()`.
3. `set_guest_notifiers()` — creates ioeventfd and irqfd pairs for each
   virtqueue, wiring them into KVM.
4. `vhost_net_start_one()` — called per queue, issues the remaining ioctls
   (VHOST_SET_VRING_KICK, VHOST_SET_VRING_ADDR, VHOST_NET_SET_BACKEND) to
   fully connect the data path.

## Kernel Module: drivers/vhost/net.c

### Device Registration

`vhost-net` registers as a misc device at `/dev/vhost-net` with file operations
defined in `vhost_net_fops`:

- `open` → `vhost_net_open()`
- `ioctl` → `vhost_net_ioctl()`
- `release` → `vhost_net_release()`

### vhost_net_open()

When QEMU opens `/dev/vhost-net`:

1. Allocates a `vhost_net` structure.
2. Sets the kick handler callbacks: `handle_tx_kick` for the TX queue and
   `handle_rx_kick` for the RX queue.
3. Calls `vhost_dev_init()` — initializes the generic vhost device
   infrastructure (mutex, work list, memory tracking).
4. Calls `vhost_poll_init()` for both TX and RX — sets up poll structures that
   will later monitor the kick eventfds.

## Kernel Data Structures

### vhost_dev (kernel side)

- `memory` — pointer to the guest memory region table (set by
  `VHOST_SET_MEM_TABLE`).
- `mm` — cached `mm_struct` of the QEMU process, used to access guest memory.
- `mutex` — serializes ioctl operations.
- `vqs` — array of pointers to `vhost_virtqueue` structures.
- `worker` — `task_struct` of the dedicated kernel thread.
- `work_list` — list of pending `vhost_work` items for the worker thread.

### vhost_virtqueue

- `desc`, `avail`, `used` — pointers into guest memory for the three virtqueue
  descriptor areas.
- `kick` — eventfd signaled by the guest to notify the host (guest-to-host).
- `call` — eventfd signaled by vhost to inject an interrupt into the guest
  (host-to-guest).
- `error` — eventfd for error reporting.
- `handle_kick` — callback invoked when the kick eventfd fires.
- `avail_idx` — tracks the last consumed available-ring index.

### vhost_net (kernel side)

- `dev` — embedded `vhost_dev`.
- `vqs[2]` — TX and RX virtqueues.
- `poll[2]` — `vhost_poll` structures for monitoring the TX and RX kick
  eventfds.

## vhost_dev_set_owner()

This ioctl is pivotal. It performs two critical operations:

1. **Records the QEMU process address space**: `dev->mm = get_task_mm(current)`.
   This captures the `mm_struct` of the calling QEMU process.

2. **Creates the vhost worker thread**: `kthread_create(vhost_worker, dev,
   "vhost-%d")`. The kernel thread name includes the PID for identification.

### The vhost_worker() Function

The worker thread is the heart of the vhost data plane:

1. Calls `use_mm(dev->mm)` to **adopt QEMU's address space**. This is the key
   trick — the kernel thread can now directly access guest memory through
   QEMU's page tables, without any copying or address translation.
2. Enters a main loop:
   - Checks `work_list` for pending `vhost_work` items.
   - Dequeues each work item and calls `work->fn()` (e.g., `handle_tx_kick`
     or `handle_rx_kick`).
   - If no work is pending, calls `schedule()` to sleep until woken.

By borrowing QEMU's mm, the vhost kernel thread can use standard kernel memory
access functions (`copy_from_user` / `copy_to_user`) to read and write guest
memory mapped into QEMU's virtual address space.

## Ioctl Interface

The vhost ioctl interface is organized into three categories:

### vhost-net Specific

| Ioctl | Purpose |
|-------|---------|
| `VHOST_NET_SET_BACKEND` | Passes the TAP device file descriptor to the kernel module, connecting vhost to the actual network backend. |
| `VHOST_GET_FEATURES` | Queries supported virtio feature bits from the kernel. |

### Device-Level

| Ioctl | Purpose |
|-------|---------|
| `VHOST_SET_OWNER` | Associates the device with the calling process; creates the worker thread. |
| `VHOST_SET_MEM_TABLE` | Communicates the guest physical memory layout via `vhost_memory` / `vhost_memory_region` structures. |
| `VHOST_SET_LOG_BASE` | Sets the base address of the dirty page bitmap for [vm-live-migration](vm-live-migration.md). |
| `VHOST_SET_LOG_FD` | Provides an eventfd to signal when dirty logging updates are available. |

### Vring-Level

| Ioctl | Purpose |
|-------|---------|
| `VHOST_SET_VRING_NUM` | Sets the virtqueue size (number of descriptors). |
| `VHOST_SET_VRING_BASE` | Sets the initial available-ring index. |
| `VHOST_SET_VRING_ADDR` | Provides guest-physical addresses of desc/avail/used areas. |
| `VHOST_SET_VRING_KICK` | Registers the guest-to-host notification eventfd (kick). |
| `VHOST_SET_VRING_CALL` | Registers the host-to-guest notification eventfd (call). |

## Notification Flow: ioeventfd and irqfd

### Guest-to-Host (TX path example)

```
Guest virtio-net driver
  └─ writes to virtio notify MMIO register
       └─ KVM traps the write via ioeventfd (no VM exit to QEMU)
            └─ signals kick eventfd
                 └─ vhost_poll_wakeup()
                      └─ queues vhost_work onto work_list
                           └─ vhost_worker wakes up
                                └─ calls handle_tx_kick()
                                     └─ reads descriptors from avail ring
                                          └─ sends packet via TAP fd
                                               └─ updates used ring
                                                    └─ signals call eventfd
```

### Host-to-Guest (completion)

```
vhost updates used ring
  └─ signals call eventfd
       └─ KVM irqfd mechanism
            └─ injects virtual interrupt into guest
                 └─ guest virtio-net driver processes completion
```

Both paths completely bypass QEMU. The ioeventfd mechanism
(see [kvm-interrupt-virtualization](kvm-interrupt-virtualization.md)) allows KVM to intercept specific
MMIO writes without exiting to userspace. The irqfd mechanism allows kernel code
to inject interrupts into the guest without involving QEMU.

## Performance Implications

The vhost-net architecture eliminates multiple context switches per packet:

- **Without vhost**: guest → KVM → QEMU (process packet) → KVM → guest.
  Each transition involves a full context switch, TLB flush, and potential
  cache pollution.
- **With vhost**: guest → KVM → vhost kernel thread (process packet) → KVM →
  guest. The vhost kernel thread runs in kernel context with QEMU's address
  space, avoiding the user/kernel transitions entirely.

The `use_mm()` trick is central to performance: by borrowing QEMU's page tables,
the vhost thread can access guest memory without building a separate mapping or
performing explicit address translation.

## See also

- [virtio-framework](virtio-framework.md) — the virtio device model that vhost accelerates
- [kvm-interrupt-virtualization](kvm-interrupt-virtualization.md) — ioeventfd and irqfd mechanisms
- [vm-live-migration](vm-live-migration.md) — dirty page tracking via VHOST_SET_LOG_BASE
- [qemu-kvm-overview](qemu-kvm-overview.md) — general QEMU/KVM architecture
