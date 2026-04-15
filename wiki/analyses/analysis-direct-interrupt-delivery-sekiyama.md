---
type: analysis
created: 2026-04-11
updated: 2026-04-11
sources: [lcna-co2012-sekiyama, vmexit-opt-hitachi-sekiyama]
tags: [kvm, direct-interrupt-delivery, cpu-isolation, vm-exit, apicv, real-time, vmcs, nmi, x2apic, interrupt-virtualization]
---

# 直接中断投递：Sekiyama 方案深度分析

> APICv 的"软件先驱"——在硬件辅助中断虚拟化出现之前，如何通过软件手段实现中断路径的零 VM Exit。

## 一、问题定义

在传统 KVM 虚拟化架构中，每一次物理中断都伴随着昂贵的 VM Exit/VM Entry 上下文切换。以一次设备中断为例，完整路径涉及 **3 组 VM Exit/Entry 往返**：

```
  Guest           Host/KVM           Guest           Host/KVM           Guest
    │                                  │                                  │
    │  ① 编程 APIC Timer               │                                  │
    │──── VM Exit ────→│               │                                  │
    │                  │ 仿真 APIC     │                                  │
    │←── VM Enter ─────│               │                                  │
    │                                  │                                  │
    │  ② Timer 到期 / 设备中断         │                                  │
    │──── VM Exit ────────────→│       │                                  │
    │                          │ IRQ   │                                  │
    │                          │ 处理  │                                  │
    │                          │ vIRQ  │                                  │
    │                          │ 注入  │                                  │
    │←── VM Enter ─────────────│       │                                  │
    │                                  │  Guest IRQ Handler               │
    │                                  │  ③ 写 EOI                        │
    │                                  │──── VM Exit ────→│               │
    │                                  │                  │ 仿真 APIC EOI │
    │                                  │←── VM Enter ─────│               │
```

**性能代价**：

- 每次 VM Exit/Entry：数百到数千个 CPU 周期（寄存器保存/恢复、TLB 刷新、流水线清空）
- HZ=1000 的 periodic timer：**每秒 3000 次 VM Exit 仅用于 timer 中断**
- 10Gb NIC passthrough 设备在高吞吐下：设备中断走同样的三段式路径
- 生产实测（ByteDance, 72-104 vCPU VM）：IPI 一项就产生 250K-550K 次 VM Exit / 5 分钟

**核心矛盾**：VMCS 中 Pin-Based Controls 的 "External interrupt exiting" 位（bit 0）默认置 1，导致所有外部中断必须经过 Host 中转。这在安全性和灵活性上有意义，但在性能上代价极高。

## 二、Sekiyama 方案：六步构建直达中断通道

Tomoki Sekiyama（日立横浜研究所, Linux Technology Center）在 LinuxCon North America 2012 上提出了一套不依赖新硬件特性的中断 VM Exit 消除方案。其核心思路是：**既然中断经过 Host 很贵，那就不让中断经过 Host**。

该方案通过六个环环相扣的步骤，构建了一条设备到 Guest 的"直达"中断通道：

### 步骤 A：CPU 隔离（CPU Isolation）

```
    ┌─────────────────────────────────────────────┐
    │              Host Linux Scheduler            │
    │                                              │
    │  pCPU 0    pCPU 1    pCPU 2    pCPU 3       │
    │  [Host]    [Host]    [─────]   [─────]      │
    │  tasks     tasks     offline   offline       │
    └─────────────────────────────────────────────┘
```

**操作**：通过 sysfs 接口将目标物理 CPU 从 Host 调度器中移除：

```bash
echo 0 > /sys/devices/system/cpu/cpu2/online
echo 0 > /sys/devices/system/cpu/cpu3/online
```

**目的**：确保该 CPU 不再运行 Host 的任何普通进程、内核线程或中断处理函数。这是后续所有步骤的前提——只有 CPU 被完全隔离，才能安全地把中断控制权交给 Guest。

### 步骤 B：绑定独占 CPU（KVM_SET_SLAVE_CPU）

**机制**：这是 Hitachi 特有的 KVM 内核补丁（非 Linux 上游主线代码）。通过新增的 ioctl 接口绑定 vCPU 到被隔离的 pCPU：

```c
ioctl(vcpu_fd, KVM_SET_SLAVE_CPU, slave_cpu_id);
```

**结果**：建立 **1:1 的 vCPU 到 pCPU 独占映射**：

```
    vCPU 0 ←──────→ pCPU 2  (独占, 禁止迁移)
    vCPU 1 ←──────→ pCPU 3  (独占, 禁止迁移)
```

- vCPU 线程在 Host 侧挂起（suspend），只在需要 QEMU 仿真的 VM Exit 时恢复
- Guest 运行期间，该 pCPU 不参与 Host 调度
- 为中断直接投递提供物理基础——中断投递目标是确定的物理 CPU

### 步骤 C：关闭 External Interrupt Exiting（核心操作）

**VMCS 设置**：修改 VMCS 的 Pin-Based VM-Execution Controls，清除第 0 位：

```
Pin-Based VM-Execution Controls:
  Bit 0: External interrupt exiting  = 0  ← 关键修改
  Bit 3: NMI exiting                 = 1  ← 保持开启（步骤 E 需要）
  Bit 5: Virtual NMIs                = ...
  Bit 6: Activate VMX preemption timer = ...
```

**行为改变**：

```
  修改前:  中断到来 → VM Exit → Host 处理 → 注入 → Guest 处理
  修改后:  中断到来 → CPU 直接根据 Guest IDT 跳转到 Guest 中断处理函数
```

**本质**：剥夺 Host 对普通中断的拦截权。从硬件视角看，外部中断不再属于"需要退出到 hypervisor 处理"的事件类别，而是直接按照当前执行环境（Guest）的 IDT 进行分发。

### 步骤 D：中断路由（IRQ Affinity）

关闭 External Interrupt Exiting 后，所有到达该 CPU 的中断都会被 Guest 处理。这要求精确控制哪些中断能到达该 CPU：

```
                    IRQ Affinity 配置
    ┌─────────────────────────────────────────────┐
    │                                              │
    │  Host 设备中断                               │
    │  (时钟, 键盘, 磁盘...)                       │
    │  ──→ smp_affinity ──→ pCPU 0, pCPU 1        │
    │                       (非隔离 CPU)           │
    │                                              │
    │  Passthrough 设备中断                        │
    │  (SR-IOV 网卡 MSI/MSI-X)                     │
    │  ──→ smp_affinity ──→ pCPU 2                │
    │                       (隔离 CPU = Guest)     │
    │                                              │
    └─────────────────────────────────────────────┘
```

**配置方式**：利用 Linux 的 `/proc/irq/<N>/smp_affinity` 或 `irqbalance` 黑名单机制。

**限制**：仅支持 MSI/MSI-X 中断（可按向量单独设置 affinity）。传统的共享 ISA IRQ 线无法做到按设备路由，仍需 Host 转发。

**关键风险**：如果 Host 的中断（如时钟中断）被意外路由到隔离 CPU，Guest IDT 中没有对应的处理函数，该中断会被**静默忽略**。如果该中断是系统关键中断，可能导致 **Host 系统卡死**。

### 步骤 E：Host 通信回退（NMI 替代）

**问题**：Host 仍然需要在某些场景下与隔离 CPU 通信：

- TLB Shootdown IPI（其他 CPU 修改了页表，需要通知该 CPU 刷新 TLB）
- Reschedule IPI（需要抢占 Guest）
- Virtual IRQ 注入（仿真设备产生的中断需要投递给 Guest）

但普通 IPI 已经无法打断 Guest（因为关闭了 External Interrupt Exiting）。

**解决方案**：利用 NMI（Non-Maskable Interrupt）。NMI 的退出行为由 VMCS Pin-Based Controls 的 bit 3（NMI exiting）独立控制，不受 bit 0 的影响。

```
  Host 需要联系隔离 CPU:
    Host ──→ 发送 NMI ──→ 隔离 CPU
                           │
                           ├─ NMI Exiting = 1
                           │
                           ▼
                        VM Exit (reason: NMI)
                           │
                           ▼
                     KVM NMI Handler
                       检查 pending request:
                       ├─ TLB Shootdown?  → 刷新 TLB
                       ├─ Reschedule?     → 调度其他 vCPU
                       └─ vIRQ Pending?   → 注入虚拟中断
```

**权衡**：NMI 是一个"重量级"通信手段（不可屏蔽，影响 NMI 嵌套处理），但在 CPU 隔离场景下使用频率低——绝大多数时间 Guest 不需要被 Host 打断。

### 步骤 F：直接 EOI（x2APIC Passthrough）

**问题**：即使中断投递零 VM Exit，Guest 处理完中断后写 EOI（End of Interrupt）寄存器仍然会触发 VM Exit（因为 APIC 寄存器访问被 KVM 拦截仿真）。

**解决方案**：利用 x2APIC 硬件的 MSR 访问模式。x2APIC 将 APIC 寄存器映射为 MSR（Model Specific Registers），而 VT-x 的 MSR Bitmap 可以控制哪些 MSR 对 Guest 直接暴露：

```
VMCS MSR Bitmap:
  MSR 0x80B (x2APIC EOI):  Read = intercept,  Write = passthrough ← 允许 Guest 直接写
```

**结果**：Guest 直接写入物理 APIC 的 EOI 寄存器，无需 VM Exit。

**注意事项**：Direct EOI 不能用于虚拟中断（emulated device 产生的中断）。虚拟中断的 EOI 必须到达 KVM 仿真的 virtual APIC，否则 KVM 无法追踪中断状态。因此：

- 注入虚拟 IRQ 时：临时**禁用** Direct EOI（恢复 MSR 拦截）
- 虚拟 IRQ 处理完毕后：重新**启用** Direct EOI

## 三、中断路径对比

### 传统 KVM（3 次 VM Exit）

```
设备中断 → pCPU → VM Exit → Host IRQ Handler → vIRQ 注入 → VM Enter
→ Guest IRQ Handler → EOI → VM Exit → APIC 仿真 → VM Enter
```

### Sekiyama 方案：Passthrough 设备中断（0 次 VM Exit）

```
设备 MSI → pCPU → Guest IDT → Guest IRQ Handler → Direct EOI
                  (无 VM Exit)                    (无 VM Exit)
```

### Sekiyama 方案：虚拟设备中断（2 次 VM Exit）

```
Host → NMI → VM Exit → QEMU 仿真 → vIRQ 注入 → VM Enter
→ Guest IRQ Handler → EOI → VM Exit → APIC 仿真 → VM Enter
→ 重新启用 Direct EOI
```

### 对比总结

| 维度 | 传统 KVM | Sekiyama 直接投递 | APICv/Posted Interrupts |
|:-----|:---------|:-----------------|:----------------------|
| **Passthrough 设备中断路径** | 设备→pCPU→VM Exit→Host→注入→Guest | 设备→pCPU→**Guest IDT 直达** | 设备→pi_desc→硬件合并→Guest |
| **VM Exit 次数（每中断）** | 3 | **0**（passthrough）/ 2（虚拟） | **0** |
| **EOI 开销** | VM Exit | 无（Direct EOI） | 无（硬件 virtual APIC） |
| **CPU 占用** | Host/KVM 消耗大量 cycle | 几乎全部给 Guest | 几乎全部给 Guest |
| **中断延迟** | 高（微秒级） | 极低（接近裸机） | 极低（接近裸机） |
| **CPU 超分配** | 支持 | **不支持**（1:1 独占） | 支持 |
| **需要新硬件** | 否 | **否** | **是**（APICv/AVIC） |
| **上游合并** | 标准 | **否**（非标准 ioctl） | 是 |

## 四、优缺点深度分析

### 优点

1. **极致性能**：Passthrough 设备中断延迟接近裸机水平，CPU 算力几乎完全属于 Guest。特别适合网络转发、高频交易、工业控制等对延迟极度敏感的场景。

2. **硬件无关**：不需要 CPU 支持 APICv、Posted Interrupts 等 2012 年后才出现的硬件特性。仅需基本的 VT-x 支持（VMCS Pin-Based Controls 和 MSR Bitmap），2006 年之后的 Intel CPU 均具备。

3. **CPU 利用率**：Host OS 在隔离 CPU 上几乎不消耗 cycle。传统方案中 KVM 处理 VM Exit 的 CPU 开销被完全消除。

### 缺点与限制

1. **兼容性差**
   - `KVM_SET_SLAVE_CPU` 是 Hitachi 私有 ioctl，非 Linux 上游主线代码
   - 修改 KVM 内核模块，与上游版本维护冲突
   - 其他 KVM 管理工具（libvirt, virsh）无法直接使用
   - 难以移植到其他 hypervisor

2. **资源浪费**
   - 必须牺牲整个物理 CPU 给单个 vCPU，**无法超卖（Overcommit）**
   - 在公有云等需要高密度部署的场景下完全不可行
   - N 个 vCPU 的 Guest 需要 N 个独占 pCPU + Host 自身的 CPU

3. **调试困难**
   - 中断不经过 Host，Host **无法监控** Guest 内部的中断状态
   - 传统的 KVM tracing 工具（kvm_stat, ftrace 的 kvm 事件）对 passthrough 中断不可见
   - Guest 内部的中断风暴或中断丢失难以从 Host 侧诊断
   - 唯一的外部干预手段是 NMI

4. **配置复杂且危险**
   - IRQ affinity 必须精确配置，否则 Host 关键中断（如时钟中断）路由到隔离 CPU 会被 Guest 静默忽略
   - 错误配置可能导致 **Host 系统卡死**（关键中断无人处理）
   - 需要管理员深入理解中断拓扑、MSI/MSI-X 向量分配、IOAPIC 路由
   - 向量号冲突处理复杂——Host 和 Guest 使用不同的 vector numbering

## 五、目标场景

Sekiyama 方案瞄准的是 **CPU 可以独占、对延迟极度敏感** 的特定工业场景：

| 场景 | 特点 | 为什么适合 |
|------|------|-----------|
| **工厂自动化 / 社会基础设施控制** | 低延迟、硬截止时间、通常单核、生命周期 10+ 年 | 单 vCPU 独占 pCPU 无压力，确定性延迟是刚需 |
| **嵌入式系统 / 设备** | RTOS Guest 与 Linux 并行运行，渐进迁移 | CPU 隔离保证 RTOS Guest 不受 Host 干扰 |
| **企业系统（自动化交易）** | 在新硬件上保留遗留软件 | 极低中断延迟是竞争优势 |
| **HPC** | 类似的延迟需求，云部署 | 计算密集型任务不容忍中断处理开销 |

**不适合的场景**：公有云（需要超分配）、多租户环境（安全隔离依赖 Host 中断拦截）、通用服务器虚拟化（管理灵活性优先）。

## 六、历史定位与现代演进

### 作为 APICv 的软件先驱

Sekiyama 方案（2012）在时间线上恰好位于 APICv 大规模部署之前。它用**纯软件手段**验证了一个关键假设：**中断可以绕过 hypervisor 直接投递给 Guest，且性能接近裸机**。

```
时间线:
  2005-2006  VT-x/AMD-V          CPU 虚拟化
  2008       EPT/NPT              内存虚拟化
  2012       Sekiyama 方案        中断直接投递 (软件方案) ←── 本文分析
  2012-2013  APICv/AVIC           中断虚拟化 (硬件方案)
  ~2020      Timer Passthrough    Timer 直通
  ~2020      NoExit PVIPI         IPI 直通
```

### 设计思想的延续

尽管 Sekiyama 方案本身已被硬件特性取代，但其三个核心设计思想在后续方案中反复出现：

| 设计思想 | Sekiyama (2012) | 现代演进 |
|----------|----------------|---------|
| **CPU 隔离** | sysfs offline + KVM_SET_SLAVE_CPU | nohz_full + isolcpus + 动态隔离（Volcengine） |
| **直接投递** | 清除 VMCS external interrupt exiting | APICv Posted Interrupts, pi_desc passthrough (NoExit PVIPI) |
| **NMI 回退** | NMI 替代 IPI 用于 Host 通信 | IPI-as-NMI（Volcengine 中断非退出方案） |

特别值得注意的是，ByteDance/Volcengine 2024 年的边缘高性能 VM 方案同样采用了 "关闭 External Interrupt Exiting + NMI 替代" 的思路——本质上是 Sekiyama 方案在 12 年后的工程再实现，但结合了 APICv、Timer Passthrough、VFIO bypass 等现代技术，并通过动态隔离解决了 CPU 独占的资源浪费问题。

### 结论

Sekiyama 方案是特定时期（缺乏硬件辅助中断虚拟化）和特定场景（Hitachi 大型机，追求极致确定性延迟）下的工程杰作。它的价值不仅在于历史贡献，更在于其设计范式对理解虚拟化中断架构的教学意义——**理解了 Sekiyama 方案的六个步骤，就理解了中断虚拟化的全部核心问题**。

## See also

- [src-lcna-co2012-sekiyama](../sources/src-lcna-co2012-sekiyama.md) — 原始 LinuxCon 2012 演讲摘要
- [src-vmexit-opt-hitachi-sekiyama](../sources/src-vmexit-opt-hitachi-sekiyama.md) — 中文技术分析文章
- [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md) — KVM 中断虚拟化全景
- [kvm-cpu-virtualization](../entities/kvm-cpu-virtualization.md) — VMCS 与 VMX 模式
- [kvm-performance-tuning](../entities/kvm-performance-tuning.md) — CPU pinning 与隔离调优
- [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md) — VM Exit 消除五代演进
- [analysis-vm-exit-reduction-and-timer-virtualization](analysis-vm-exit-reduction-and-timer-virtualization.md) — VM Exit 消除发展历程与 Timer 优化
- [analysis-timer-vmexit-optimization-survey](analysis-timer-vmexit-optimization-survey.md) — Timer VM Exit 优化方案综述
