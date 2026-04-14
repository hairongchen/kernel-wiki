---
type: source
title: "QEMU/KVM Source Code Analysis and Application"
created: 2026-04-09
updated: 2026-04-09
sources: [qemu-kvm-source-code-and-application]
tags: [qemu, kvm, virtualization, source-summary]
---

# QEMU/KVM Source Code Analysis and Application

**Author**: Li Qiang (李强)
**Publisher**: China Machine Press (机械工业出版社), 2021
**ISBN**: 9787111661160
**Language**: Chinese
**Source code versions**: QEMU 2.8.1, Linux kernel 4.4.161, SeaBIOS rel-1.11.2

A comprehensive source-code-level analysis of the QEMU/KVM virtualization stack. Covers the full system virtualization pipeline from QEMU's userspace device emulation through KVM's kernel-level hardware-assisted virtualization, with detailed treatment of CPU, memory, interrupt, and I/O virtualization. Focuses on the x86 platform with Intel VT-x.

## Chapter-by-Chapter Summary

### Chapter 1: QEMU and KVM Overview
Introduces virtualization concepts (Type 1 vs Type 2 hypervisors, full vs para-virtualization), Intel VT-x (VMX root/non-root modes, VMCS), and the QEMU/KVM architecture. QEMU provides userspace device emulation; KVM is a Linux kernel module exposing `/dev/kvm`. Three execution modes: guest (VMX non-root), kernel (VM-Exit handler), userspace (QEMU). The VM Entry/Exit loop is the core execution mechanism.

### Chapter 2: QEMU Basic Components
Covers QEMU's internal infrastructure. **QOM** (QEMU Object Model): TypeInfo/TypeImpl/ObjectClass/Object hierarchy, lazy class initialization, instance construction, C-language polymorphism via embedded parent structs and function-pointer vtables. **QDev**: DeviceClass/DeviceState/BusState device model with realize/unrealize lifecycle, child/link properties. **Event loop**: GLib-based main loop, AioContext, bottom halves, timers. **Thread model**: main thread + per-VCPU threads, Big QEMU Lock (BQL). **HMP/QMP**: Human Monitor Protocol (text) and QEMU Machine Protocol (JSON), QAPI schema system with code generation.

### Chapter 3: Motherboard and Firmware Emulation
Intel 440FX chipset emulation: i440FX northbridge (PCI host bridge, memory controller) + PIIX3/PIIX4 southbridge (ISA bridge, IDE, USB, power management). Device tree structure rooted at `main-system-bus`. Machine type initialization via `DEFINE_I440FX_MACHINE` macro and `pc_init1()`. **fw_cfg** device for QEMU-to-firmware communication. **SeaBIOS** boot flow: POST, PCI scan, boot device selection. ACPI table generation (RSDP, FADT, DSDT, MADT, HPET, MCFG).

### Chapter 4: CPU Virtualization
**QEMU side**: x86 CPU as QOM object (`X86CPU`/`CPUX86State`), feature negotiation, `x86_cpu_realizefn` triggers VCPU thread creation. `kvm_cpu_exec()` loop: `KVM_RUN` ioctl, VM-Exit handling (`KVM_EXIT_IO`, `KVM_EXIT_MMIO`). CPU reset sets `CS:EIP = 0xF000:0xFFF0`. **KVM side**: `KVM_CREATE_VCPU` ioctl, `vmx_create_vcpu()` allocates VMCS and VPID, `vmx_vcpu_setup()` initializes VMCS fields. QEMU-KVM shared data via mmap'd `kvm_run` structure.

### Chapter 5: Memory Virtualization
**KVM memory slots**: `kvm_memory_slot` with GFN-to-HVA mapping, `KVM_SET_USER_MEMORY_REGION` ioctl. **EPT (Extended Page Tables)**: GVA→GPA→HPA two-level translation. EPT violation handling: `handle_ept_violation()` → `tdp_page_fault()` → `gfn_to_pfn()` + `__direct_map()`. **Shadow page tables** for non-EPT mode. **MMU notifiers** for host page table change tracking. **Dirty page tracking**: write-protection or Intel PML (Page Modification Logging), `KVM_GET_DIRTY_LOG` ioctl.

### Chapter 6: Interrupt Virtualization
**PIC** (8259): dual cascaded controllers, IRR/ISR/IMR registers. **I/O APIC**: 24-pin redirection table, MSI message formatting. **Local APIC**: per-VCPU, LVT, ICR, timer (one-shot/periodic/TSC-deadline). **MSI**: direct LAPIC delivery bypassing I/O APIC. **APICv**: hardware-assisted APIC virtualization — virtual-interrupt delivery, APIC-register virtualization, posted interrupt descriptors (PIR). **IRQfd**: eventfd→GSI injection. **IOeventfd**: guest I/O write→eventfd signaling for doorbell notifications.

### Chapter 7: Device and I/O Virtualization
**Device model**: BusClass/BusState, DeviceClass/DeviceState, `qdev_create()`/`device_set_realized()`. **Virtio framework**: VRing (descriptor table, available ring, used ring), VirtQueue, VirtIODevice base class, PCI transport (`VirtIOPCIProxy`). **Virtio-net**: `VirtIONet`, TX path (`virtio_net_flush_tx`→`qemu_sendv_packet`), RX path (`virtio_net_receive`→`virtqueue_fill`), mergeable RX buffers, TAP backend, multiqueue. **Virtio-blk**: `VirtIOBlock`/`VirtIOBlockReq`, `blk_aio_preadv/pwritev`, IOThread data plane. Other: virtio-balloon, virtio-serial, virtio-rng, virtio-gpu. **VGA emulation**: `VGACommonState`, register I/O, planar memory model, VBE extensions. Display pipeline: `DisplaySurface`→`QemuConsole`→`DisplayChangeListener` (SDL/GTK/VNC/Spice). **Vhost**: kernel-based data plane offloading via `/dev/vhost-net`, `vhost_worker` kernel thread, ioeventfd/irqfd notification paths bypass QEMU entirely.

### Chapter 8: Other Topics
**VM live migration**: `migration_thread` with 4 phases (setup, iterate, pending, complete). `SaveStateEntry` with `SaveVMHandlers` (live data) or `VMStateDescription` (device state). RAM migration: dirty bitmap, `memory_global_dirty_log_start()`, `KVM_MEM_LOG_DIRTY_PAGES`, iterative dirty page sending, convergence via bandwidth/downtime control, VCPU throttling. **XBZRLE** compression for incremental page transfer. **Post-copy** migration via `userfaultfd`. **VMState**: serialization framework with `VMSTATE_*` macros. **Security**: device emulation as primary attack surface, VM escape risks, SMBus/VNC vulnerability examples. Mitigation: Intel nemu (reduced QEMU), Google crosvm / Amazon Firecracker (Rust-based VMMs), seccomp sandboxing. **Containers**: Docker, gVisor (Sentry kernel + seccomp), convergence of VMs and containers.

## Key Themes

1. **QEMU/KVM split**: QEMU handles control plane (device emulation, configuration) in userspace; KVM handles data plane (CPU execution, memory mapping) in kernel. Communication via ioctl and shared mmap'd structures.
2. **Object-oriented C**: QOM provides inheritance, polymorphism, and properties via embedded structs and function-pointer vtables — a consistent pattern used throughout QEMU.
3. **Hardware-assisted virtualization**: VT-x (VMX), EPT, APICv, and posted interrupts progressively eliminate VM-Exit overhead. The book traces how each hardware feature maps to KVM code.
4. **Virtio as the paravirtualization standard**: VRing shared-memory protocol enables efficient guest-host I/O. Vhost moves the hot path into the kernel, and ioeventfd/irqfd eliminate QEMU from notification paths entirely.
5. **Live migration as a design constraint**: Every emulated device must implement VMState serialization. RAM migration uses iterative dirty page tracking with convergence control.
6. **Security through reduction**: The trend from full QEMU toward stripped-down VMMs (nemu, Firecracker) and memory-safe languages (Rust) reflects the security cost of device emulation complexity.

## Relationship to Other Sources

This book complements the existing wiki sources by covering **virtualization** — a topic not addressed by the other five books. It extends the kernel knowledge in several directions: KVM uses the same kernel subsystems (memory management, interrupts, scheduling) documented in "Understanding the Linux Kernel" and "Professional Linux Kernel Architecture". The virtio networking stack connects to the networking internals covered in "Understanding Linux Network Internals". The `/dev/kvm` character device follows the device model patterns from "Advanced Linux Programming".

## See also

- [qemu-kvm-overview](../entities/qemu-kvm-overview.md)
- [kvm-cpu-virtualization](../entities/kvm-cpu-virtualization.md)
- [kvm-memory-virtualization](../entities/kvm-memory-virtualization.md)
- [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md)
- [virtio-framework](../entities/virtio-framework.md)
- [vhost](../entities/vhost.md)
- [vm-live-migration](../entities/vm-live-migration.md)
- [qemu-machine-emulation](../entities/qemu-machine-emulation.md)
