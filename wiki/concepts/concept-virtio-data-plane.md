---
type: concept
created: 2026-04-09
updated: 2026-04-09
sources: [qemu-kvm-source-code-and-application]
tags: [virtio, vhost, data-plane, ioeventfd, irqfd, performance]
---

# Virtio Data Plane

The performance of I/O virtualization in QEMU/KVM has been shaped by a single organizing principle: separate the data plane from the control plane, then push the data plane as close to the hardware as possible. Each successive optimization removes a layer of indirection from the I/O notification and processing path, reducing latency and increasing throughput.

## Three Levels of I/O Virtualization

### Level 1: Full Device Emulation

In full emulation, QEMU presents the guest with a virtual device that mimics a real hardware device (e.g., an Intel e1000 NIC or IDE disk controller). The guest runs its standard, unmodified driver for that device.

**How it works:**

1. The guest driver writes to a device register (a memory-mapped I/O or port I/O address).
2. This triggers a VM Exit from VMX non-root mode to KVM in the host kernel.
3. KVM determines the exit is for an I/O operation and returns to QEMU userspace.
4. QEMU's device emulation code processes the operation (e.g., reads from a disk image file, writes to a TAP network interface).
5. When the operation completes, QEMU injects an interrupt into the guest via KVM.
6. KVM performs a VM Entry to resume guest execution with the interrupt pending.

**Performance cost:** Every single I/O register access causes a full VM Exit to userspace and back. A single network packet or disk block can involve multiple register accesses. The transitions between guest, kernel, and userspace dominate the I/O latency.

### Level 2: Paravirtualization with Virtio

Virtio replaces hardware device emulation with a standardized paravirtual interface. The guest knows it is in a VM and uses purpose-built virtio drivers that communicate through shared memory ring buffers (VRings/virtqueues) rather than emulating hardware register accesses.

**How it works:**

1. The guest virtio driver places I/O requests into a shared-memory virtqueue (a ring buffer in guest RAM that both guest and host can access).
2. The guest writes to a single "kick" register to notify the host that new requests are available.
3. This kick causes a VM Exit to KVM, which returns to QEMU.
4. QEMU reads the requests directly from the shared virtqueue memory (no data copying through registers -- bulk data stays in place).
5. QEMU processes the requests and places completions back in the virtqueue.
6. QEMU injects an interrupt to notify the guest of completions.

**Improvement over Level 1:** Data transfer happens through shared memory rather than register-by-register emulation. One kick notification can batch multiple requests. The number of VM Exits per I/O operation drops dramatically. But QEMU userspace is still in the processing path for every I/O operation.

### Level 3: Vhost Kernel Offloading

Vhost moves the data plane -- the actual processing of virtqueue requests -- from QEMU userspace into a dedicated kernel thread (`vhost_worker`). QEMU retains only the control plane: device setup, feature negotiation, migration state management. The steady-state I/O path bypasses QEMU entirely.

**How it works:**

1. During device setup, QEMU configures the vhost kernel module with the virtqueue memory layout, file descriptors for the backend (e.g., TAP device), and eventfd descriptors for notifications.
2. At runtime, the guest places requests in the shared virtqueue and writes to the kick register.
3. Instead of routing through QEMU, this kick is delivered via **ioeventfd**: KVM translates the guest MMIO write directly into an eventfd signal without returning to userspace.
4. The kernel `vhost_worker` thread is waiting on this eventfd. It wakes, processes the virtqueue entries, and performs the actual I/O (e.g., writes to the TAP device).
5. When complete, vhost signals guest completion via **irqfd**: it writes to an eventfd that KVM monitors, and KVM injects the interrupt directly into the guest -- again without involving QEMU.

**Improvement over Level 2:** QEMU is completely removed from the data path. The notification and processing path stays entirely within the kernel. No userspace transitions for steady-state I/O.

## The Notification Path in Detail

### Without Vhost

```
Guest driver writes to kick register
  → VM Exit to KVM (kernel)
  → KVM returns to QEMU (userspace)
  → QEMU reads virtqueue from shared memory
  → QEMU performs I/O (e.g., write() to TAP fd)
  → QEMU calls KVM ioctl to inject interrupt
  → VM Entry to guest with interrupt pending
```

Transitions: guest → kernel → userspace → kernel (I/O) → kernel (KVM) → guest. QEMU sits in the middle of every operation.

### With Vhost (ioeventfd + irqfd)

```
Guest driver writes to kick register
  → VM Exit to KVM (kernel)
  → KVM matches the MMIO address to a registered ioeventfd
  → KVM signals the eventfd (no return to userspace)
  → vhost_worker thread wakes on eventfd
  → vhost_worker reads virtqueue, performs I/O to TAP
  → vhost_worker signals irqfd eventfd
  → KVM receives irqfd signal, injects interrupt into guest
  → VM Entry (or posted interrupt if guest is running)
```

Transitions: guest → kernel → kernel → guest. QEMU is not involved at all. The entire data path is kernel-to-kernel.

### Interrupt Return Path Comparison

**Without irqfd:** QEMU must call `ioctl(KVM_IRQ_LINE)` or `ioctl(KVM_SIGNAL_MSI)` to inject an interrupt. This means QEMU must be scheduled by the host kernel, context-switch into QEMU, make a syscall into KVM, and then KVM can deliver the interrupt on the next VM Entry.

**With irqfd:** The vhost_worker thread writes to an eventfd. KVM has registered a callback on this eventfd that directly calls the interrupt injection code. If the vCPU is in guest mode and the hardware supports posted interrupts (APICv), the interrupt can be delivered without even a VM Exit.

## IOThread Data Plane for virtio-blk

An intermediate optimization applies specifically to QEMU's virtio-blk device. Rather than moving the entire data plane to kernel space (as vhost does), the IOThread model moves virtqueue processing out of QEMU's main event loop and into a dedicated IOThread.

In the default configuration, QEMU runs a single main loop thread that handles all device emulation, management operations, and virtqueue processing. This creates contention: a virtqueue processing burst blocks management operations, and vice versa.

With IOThread data plane:
- Virtqueue processing runs in a separate thread with its own event loop
- The main thread handles only control plane operations
- This provides concurrency between I/O processing and management tasks
- Multiple virtio-blk devices can have their own IOThreads for parallelism

This approach is less radical than vhost (processing still happens in QEMU userspace) but simpler to implement for block devices and avoids some of the complexity of kernel-based data plane.

## ioeventfd and irqfd Mechanisms

### ioeventfd

ioeventfd is a KVM feature that maps a guest MMIO or PIO address to a Linux eventfd. When the guest writes to that address, instead of the normal VM Exit handling path (return to userspace, let QEMU process it), KVM directly signals the eventfd and optionally resumes the guest immediately.

This is registered via the `KVM_IOEVENTFD` ioctl. Key fields include:
- The guest address to monitor
- The eventfd file descriptor to signal
- An optional data value to match (so different values written to the same address can trigger different eventfds)

### irqfd

irqfd is the reverse direction: it maps a Linux eventfd to a guest interrupt. When something writes to the eventfd, KVM injects the specified interrupt into the guest.

This is registered via the `KVM_IRQFD` ioctl. Key fields include:
- The eventfd file descriptor to monitor
- The GSI (Global System Interrupt) number to inject
- Flags for controlling behavior (e.g., deassign, resample for level-triggered interrupts)

Together, ioeventfd and irqfd create a bidirectional kernel-space notification channel between the guest and a kernel-based I/O handler, completely bypassing QEMU userspace.

## The Trend: Further Data Plane Optimization

### DPDK and SPDK

DPDK (Data Plane Development Kit) and SPDK (Storage Performance Development Kit) take the opposite approach from vhost: instead of moving the data plane into the kernel, they move it into a dedicated userspace process that polls for I/O completions rather than using interrupts. By dedicating CPU cores to busy-polling device queues, they eliminate both interrupt overhead and context-switch latency. DPDK is used for high-performance networking (e.g., OVS-DPDK for virtual switching), while SPDK provides a similar model for NVMe storage.

### vDPA (virtio Data Path Acceleration)

vDPA represents the logical endpoint of the data plane optimization trajectory. With vDPA, the hardware itself implements the virtqueue interface. The guest's virtio driver communicates directly with a hardware virtqueue on the physical NIC or storage controller. The hypervisor is involved only in control plane setup. The data plane is entirely in hardware -- zero software involvement for steady-state I/O.

### Minimal VMMs and Security

The data plane / control plane separation has a security dimension. QEMU is a large, complex codebase (~2 million lines of code) that historically has had a significant attack surface. If a guest can exploit a vulnerability in QEMU's device emulation, it can escape the VM.

Reducing QEMU's role in the data path is not just a performance optimization -- it also reduces the attack surface. This insight has driven the development of minimal VMMs:

- **nemu**: A stripped-down fork of QEMU removing legacy device emulation
- **crosvm** (Chrome OS): A VMM written in Rust with minimal device emulation
- **Firecracker** (AWS): A purpose-built VMM for serverless workloads with only a handful of emulated devices

These projects reduce the code reachable by a malicious guest to the bare minimum, relying on virtio and vhost for I/O rather than complex device emulation.

## Summary of the Optimization Trajectory

| Approach | Data plane location | Notification mechanism | QEMU involvement |
|----------|-------------------|----------------------|-------------------|
| Full emulation | QEMU main thread | VM Exit to userspace | Every I/O operation |
| Virtio | QEMU main thread | VM Exit to userspace (batched) | Every I/O batch |
| Virtio + IOThread | QEMU IOThread | VM Exit to userspace | I/O processing (dedicated thread) |
| Vhost | Kernel vhost_worker | ioeventfd / irqfd | Control plane only |
| DPDK/SPDK | Dedicated userspace poll | Polling (no interrupts) | None (separate process) |
| vDPA | Hardware | Hardware virtqueue | Control plane only |

The pattern is clear: identify the hot path, then move it out of QEMU's main loop, out of QEMU entirely, out of the kernel, and ultimately into hardware.

## See also

- [virtio-framework](../entities/virtio-framework.md)
- [vhost](../entities/vhost.md)
- [vm-live-migration](../entities/vm-live-migration.md)
- [concept-hardware-virtualization](concept-hardware-virtualization.md)
- [vfio-device-passthrough](../entities/vfio-device-passthrough.md)
