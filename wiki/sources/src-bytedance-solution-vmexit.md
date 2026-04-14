---
type: source
created: 2026-04-10
updated: 2026-04-10
sources: [bytedance-solution-vmexit]
tags: [vm-exit, edge-computing, volcengine, bytedance, timer-passthrough, ipi-fastpath, vfio, interrupt-passthrough, nohz-full, bare-metal, performance]
---

# Source: 性能媲美裸金属，边缘场景高性能虚拟机技术揭秘

| Field | Value |
|-------|-------|
| Title (EN) | Performance Rivaling Bare Metal: High-Performance VM Technology for Edge Scenarios |
| Author | 付秋伟 (Fu Qiuwei), 火山引擎 (Volcengine / ByteDance) |
| Publisher | InfoQ |
| Date | 2024-12-30 |
| Coverage | Edge computing high-performance VM: complete hardware passthrough framework |

## Summary

Volcengine (ByteDance's cloud arm) describes their edge computing high-performance VM technology that achieves near-bare-metal performance by systematically eliminating VM Exits through four key techniques: interrupt non-exit, timer passthrough, VFIO interrupt bypass, and IPI extreme fastpath, combined with kernel resource dynamic isolation. The article provides the most comprehensive public description of their production architecture.

## Section-by-Section Summary

### 1. Edge Computing Background

Volcengine edge computing nodes are distributed across China, providing compute, network, storage, and security services at the network edge. Two compute forms: edge VMs (elastic, multi-architecture x86/ARM/GPU) and edge bare metal (direct hardware access). The article focuses on making edge VMs achieve bare-metal performance.

### 2. Performance Requirements: Business Cases

**Live streaming**: Hosts push streams to edge nodes, viewers pull from nearest edge. Unit compute cost per bandwidth directly affects business costs. Customers are sensitive to VM vs bare-metal performance and cost differences.

**Network acceleration**: Game/app acceleration via globally distributed edge nodes. Customers have extremely low tolerance for network jitter. Interrupts, process preemption, and VM Exits all cause jitter that degrades service quality.

### 3. Architecture: Breaking the Virtualization Boundary

**Design philosophy**: "Break the virtualization boundary" — make the Guest Kernel run as if on physical hardware. Each high-performance VM's vCPU exclusively owns a physical CPU, memory, and interrupts.

**Design goals**:
- Minimize VM Exit count to match bare metal
- Transparent to Guest (no guest image modification required)
- Support hot upgrade, live migration, and other cloud capabilities
- Support coexistence with regular VM instances

**Architecture diagram** (from article):

```
User space:
  QEMU ─────────────► High Performance VM
  Libvirt               ├── Application
                        └── Unmodified Guest Kernel
                            vCPU  vCPU  vCPU  vCPU

Kernel space:
  kernel resource      extreme fastpath IPI
  dynamic isolation    timer passth
  framework            Device IRQ passth
                       └── IRQ passth framework ──── KVM

Hardware:
  Host core  Host core  Guest core  Guest core ... Guest core
```

### 4. Key Technology: Guest Interrupt Non-Exit Mechanism

**Problem**: Traditional interrupt delivery requires VM Exit → VMM emulation → VM Entry. Even with APICv, Posted-Interrupt hardware still introduces overhead compared to bare-metal direct delivery.

**Solution**: For passthrough devices (LAPIC Timer, VFIO devices), configure VMCS `INTR_EXITING` field to prevent external interrupt VM Exits. Interrupts on the isolated pCPU are delivered directly to the Guest.

**Challenge**: Not all interrupts on the pCPU belong to the Guest (e.g., host kernel IPI, host device interrupts like NVMe SSD).

**Host interrupt isolation**:
- Device interrupts: Migrate all device interrupt affinity to designated control-plane CPUs
- System IPI: Use **"send IPI as NMI"** technique — modify IPI type to NMI, which forces vCPU kick-out to host kernel for handling

### 5. Key Technology: Timer Passthrough

**Problem**: Guest Timer is emulated by VMM. In TSC-Deadline mode, guest programming TSC Deadline MSR causes one VM Exit, timer firing causes another external interrupt VM Exit. Intel's newer APIC Timer hardware acceleration is not yet widely commercially available and itself introduces hardware overhead.

**Solution**: Force host CPU to use early Broadcast Timer mode, then pass each CPU's LAPIC Timer completely to the Guest. For passthrough Guest LAPIC Timer, configure Guest to not exit on TSC Deadline MSR write. Combined with the interrupt non-exit mechanism, Guest LAPIC Timer produces **zero VM Exits**.

### 6. Key Technology: VFIO Interrupt Passthrough Bypass

**Problem**: Traditional VFIO passthrough device interrupt injection uses Intel Posted-Interrupt mechanism, which introduces hardware overhead. When a passthrough device generates an interrupt, IOMMU intercepts it, looks up IRTE table, then uses interrupt remapping or Posted-Interrupt to inject.

**Solution**: Modify IOMMU IRTE table entries to directly record the Guest's destination vCPU and vector, so passthrough device interrupts are delivered directly to Guest internals — bypassing the Posted-Interrupt indirection layer entirely.

### 7. Key Technology: IPI Extreme Fastpath

**Problem**: Guest IPI (writing virtual LAPIC ICR) causes VM Exit for VMM emulation. VMM must emulate IPI logic and inject the interrupt to the target vCPU.

**Solution**: Do not passthrough ICR (security concern). Instead, optimize the VMM-side IPI emulation path: use **assembly-rewritten** simplified IPI logic, perform ICR safety check, then write physical ICR directly to deliver IPI to the target pCPU. This minimizes the write-ICR exiting performance overhead to the extreme.

**Note**: This builds on top of the community IPI fastpath but goes further with assembly optimization.

### 8. Key Technology: Kernel Resource Dynamic Isolation

**Problem**: To maximize Guest's exclusive use of physical cores, host timer interrupts, device interrupts, and host processes must be isolated away from Guest cores.

**Three isolation mechanisms**:

1. **Dynamic nohz_full**: Built on the kernel's `NOHZ_full` framework (which is static, requiring kernel cmdline). The high-perf VM solution implements **dynamic nohz_full** — entering `nohz_full` state on vCPU enter Guest, exiting on vCPU exit Guest. Maximally reduces host Timer interrupt impact on Guest runtime.

2. **Dynamic interrupt isolation**: Similar to kernel's `isolcpus` interrupt isolation but dynamic. Before creating high-perf VMs, migrate all host device interrupts to control-plane CPUs and prevent new interrupts from being assigned to Guest cores.

3. **Dynamic process isolation**: Similar to `isolcpus` process isolation. Before VM creation, migrate all host processes to control-plane CPUs and pin vCPU threads to prevent migration.

### 9. Benchmarks

**Micro benchmarks** (latency in ns, lower is better):
- Single IPI latency: **down ~22%** (high-perf vs. general)
- Broadcast IPI latency: **down ~17%**
- Timer latency: **down ~12.5%** (approaching bare metal)
- `MSR_IA32_TSCDEADLINE` write latency: **down ~89%**
- `MSR_IA32_POWER_CTL` write latency: **down ~89%**

**VM Exit count** (with `idle=poll`):
- Redis in 60s + Nginx in 60s: **VM Exit count reduced by >99%** compared to general instances
- General instance VM Exits concentrated in three categories: `MSR_WRITE`, `External_interrupt`, `Preemption_Timer`

**Application benchmarks** (throughput, higher is better):
- wrk (HTTP): **+6%**
- Apache: **+12%**
- Redis LRANGE_500: **+16%**
- netperf TCP_STREAM: **+8%**
- netperf UDP_STREAM: **+11%**
- netperf TCP_RR: **+13%**

**Real-world deployments**:
- **CDN**: CPU usage reduced 13.9-23.2% vs general VM, within 0.2-2.9% of bare metal
- **Live streaming**: CPU usage reduced 23.7%, performance matches bare metal
- **Network acceleration**: Latency stability matches bare metal (flat ~4ms over 8-hour window)

### 10. Conclusion and Future

Edge high-perf VM is in production on edge computing nodes. Future plans include GPU and heterogeneous compute scenarios.

## Key Themes

1. **Systematic VM Exit elimination**: Not just one technique but a coordinated framework addressing every exit source (timer, IPI, device interrupt, preemption timer)
2. **Guest transparency**: All optimizations are host-side only — unmodified guest images work directly
3. **Dynamic isolation**: Unlike static kernel parameters, isolation is applied/removed dynamically around VM lifecycle
4. **Edge-specific constraints**: Small nodes, cost-sensitive workloads, low-latency requirements make the VM-vs-bare-metal gap commercially significant
5. **Production validated**: Real CDN/streaming/acceleration workloads show measurable business impact

## Relationship to Other Sources

- Provides the production architecture behind techniques described in [src-minimizing-vmexits-pv-ipi-passthrough-timer](src-minimizing-vmexits-pv-ipi-passthrough-timer.md)
- Extends Sekiyama's CPU isolation concepts ([src-lcna-co2012-sekiyama](src-lcna-co2012-sekiyama.md)) with dynamic isolation
- Adds IPI fastpath assembly optimization and VFIO interrupt bypass not covered in other sources
- Complements [src-all-solution-vmexit](src-all-solution-vmexit.md) by providing the full ByteDance architecture context

## See also

- [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md)
- [kvm-performance-tuning](../entities/kvm-performance-tuning.md)
- [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md)
- [concept-virtio-data-plane](../concepts/concept-virtio-data-plane.md)
- [vfio-device-passthrough](../analyses/vfio-device-passthrough.md)
