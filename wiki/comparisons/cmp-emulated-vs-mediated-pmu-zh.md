---
type: comparison
created: 2026-04-10
updated: 2026-04-10
sources: [qemu-kvm-source-code-and-application, sean-jc-mediated-pmu-branch]
tags: [kvm, pmu, performance-monitoring, virtualization, perf, vpmu]
---

# 模拟 PMU 与中介 PMU 对比分析

KVM 为客户机虚拟机提供了两种架构上截然不同的 x86 性能监控单元 (PMU) 虚拟化方案：传统的**模拟 PMU**（基于宿主机 `perf` 子系统）和更新的**中介 PMU**（直接访问硬件计数器）。本文基于上游 KVM 代码库和 Sean Christopherson 的中介 PMU 补丁系列（`vmx/mediated_pmu_freeze_in_smm` 分支，lore: `20260207041011.913471-*-seanjc@google.com`），对两者的设计、权衡和实现细节进行比较。

## 概览

| 维度 | 模拟 PMU | 中介 PMU |
|------|---------|---------|
| 核心机制 | KVM 代表客户机编程宿主机 `perf_event` | KVM 将客户机 PMU 状态直接加载到硬件 MSR |
| 计数器归属 | 宿主机 perf 子系统拥有硬件计数器 | 客户机在 VM-Entry 期间"借用"硬件计数器 |
| MSR 访问 | 所有 PMU MSR 访问都会触发 VM-Exit | 大部分 PMU MSR 访问为直通（不触发 VM-Exit） |
| PMI 传递 | 通过 perf 溢出回调 → KVM 注入 | 通过硬件 PMI → `kvm_handle_guest_mediated_pmi()` |
| 精确度 | 近似值（基于采样，存在滑移） | 硬件精确（计数在真实硬件中运行） |
| 宿主-客户隔离 | 强隔离（perf 中介所有访问） | 基于上下文切换（VM-Entry/Exit 时加载/卸载） |
| 硬件要求 | PMU 版本 >= 1 | Intel PMU v4+，全宽度写入，VMCS PERF_GLOBAL_CTRL 加载 |
| 与宿主机的复用 | 自动（perf 调度事件） | 客户机执行期间独占（宿主 perf 被禁用） |

## 架构

### 模拟 PMU：以 perf 为后端

模拟 PMU 将客户机的 PMU 视为一个*虚拟设备*。当客户机编程性能计数器（写入 `IA32_PERFEVTSELx`、`IA32_PMCx` 等）时，每次 MSR 访问都会触发 VM-Exit。KVM 拦截这些写入并将其转换为宿主机 `perf_event` 操作：

```
客户机写入 MSR_P6_EVNTSEL0
  → VM-Exit（MSR 写入）
  → kvm_pmu_set_msr()
    → reprogram_counter()
      → pmc_pause_counter()           // 暂停现有 perf_event
      → pmc_reprogram_counter()       // 创建新 perf_event
        → perf_event_create_kernel_counter()
          attr.exclude_host = 1        // 仅在客户机上下文中计数
          attr.sample_period = f(pmc->counter)
```

关键特征：

- **每个计数器操作都经过 perf**：读取计数器（`RDPMC`、`RDMSR`）时通过 `perf_event_pause()` 累积计数。写入计数器时调整相对于 perf 事件当前计数的偏移量。
- **溢出检测基于采样**：当计数器跨越零值时，perf 生成中断（基于 `sample_period`）。溢出回调 `kvm_perf_overflow()` 在 `global_status` 中设置相应位并请求向客户机注入 PMI。
- **模拟指令计数**：对于 KVM 模拟的指令（perf 无法观察到的），KVM 维护一个单独的 `pmc->emulated_counter`，在 `pmc_pause_counter()` 期间合并到总计数中。

```
pmc_pause_counter():
  counter += perf_event_pause(perf_event, true)   // 硬件计数
  counter += pmc->emulated_counter                 // 软件计数
  pmc->counter = counter & bitmask
  pmc->emulated_counter = 0
```

- **事件调度**：宿主机 perf 子系统拥有物理计数器。如果宿主机也要使用 PMU 计数器，perf 会在宿主和客户事件之间进行时分复用。交叉映射检测（`intel_pmu_cross_mapped_check()`）处理客户和宿主事件竞争相同物理计数器的情况。

### 中介 PMU：直接硬件访问

中介 PMU 赋予客户机对物理 PMU 硬件的*直接访问*能力。KVM 不再通过 perf 转换客户机的 PMU 操作，而是在 VM-Entry 时将客户机的原始 PMU MSR 值加载到硬件中，并在 VM-Exit 时将其读回：

```
VM-Entry 路径：
  kvm_mediated_pmu_load()
    → perf_load_guest_context()        // 通知 perf 让出 PMU
    → wrmsrq(PERF_GLOBAL_CTRL, 0)     // 禁用所有计数器
    → perf_load_guest_lvtpc(APIC_LVTPC) // 设置 PMI 传递向量
    → kvm_pmu_load_guest_pmcs()        // 将客户机计数器和选择器写入硬件
    → intel_mediated_pmu_load()        // 同步 GLOBAL_STATUS、FIXED_CTR_CTRL
    → vmcs_write64(GUEST_IA32_PERF_GLOBAL_CTRL, global_ctrl)
                                       // VM-Entry 时原子启用

VM-Exit 路径：
  kvm_mediated_pmu_put()
    → intel_mediated_pmu_put()         // 读回 GLOBAL_STATUS，清除硬件状态
    → kvm_pmu_put_guest_pmcs()         // 通过 rdpmc() 将所有计数器读回 pmc->counter
    → perf_put_guest_lvtpc()           // 恢复宿主机 PMI 向量
    → perf_put_guest_context()         // 将 PMU 控制权归还宿主 perf
```

关键特征：

- **MSR 直通**：当客户机拥有与宿主硬件相同数量的计数器时，PMU MSR 访问不会触发 VM-Exit。客户机以原生速度读写计数器和事件选择器。这由 `vmx_recalc_pmu_msr_intercepts()` 中的 MSR 位图控制。
- **计数器值为物理值**：`pmc->counter` 直接反映硬件计数器的值。写入操作十分简单：`pmc->counter = val & pmc_bitmask(pmc)`。无需 perf 事件、无需偏移量运算、无需模拟计数器。
- **PERF_GLOBAL_CTRL 通过 VMCS 传递**：在 Intel 上，`PERF_GLOBAL_CTRL` 通过专用 VMCS 字段（`VM_ENTRY_LOAD_IA32_PERF_GLOBAL_CTRL`、`VM_EXIT_SAVE_IA32_PERF_GLOBAL_CTRL`）原子加载/保存。这消除了客户机事件选择器在宿主全局控制下活动（或反之）的窗口。
- **PMI 为硬件原生**：当中介计数器溢出时，硬件生成真实的 PMI。NMI/PMI 处理程序调用 `kvm_handle_guest_mediated_pmi()`，该函数简单地在 vCPU 上设置 `KVM_REQ_PMI`。无需 perf 事件回调。
- **模拟指令计数为直接操作**：对于模拟指令，中介路径简单地递增 `pmc->counter` 并检查回绕。不需要 `emulated_counter` 累加器。

```
kvm_pmu_incr_counter() [中介路径]:
  pmc->counter = (pmc->counter + 1) & pmc_bitmask(pmc)
  if (!pmc->counter) {
      pmu->global_status |= BIT_ULL(pmc->idx)
      if (pmc_is_pmi_enabled(pmc))
          kvm_make_request(KVM_REQ_PMI, vcpu)
  }
```

## 详细对比

### 1. VM-Exit 开销

**模拟 PMU**：每次 PMU MSR 访问都会触发 VM-Exit。对于频繁读写性能计数器的工作负载（例如客户机内运行 `perf stat` 或性能分析守护进程），这会产生显著的开销。仅状态保存/恢复就需要数百到数千个 CPU 周期，还要加上内核侧的处理时间。

**中介 PMU**：当客户机拥有完整的硬件计数器集时，PMU MSR 访问为直通。`RDPMC`、对 `PERFCTRx`/`PERFEVTSELx`/`FIXED_CTRx` 的 `RDMSR`/`WRMSR` 均以原生速度执行。仅在以下情况需要拦截：
- 客户机拥有的计数器少于硬件（防止访问不存在的计数器）
- `PERF_GLOBAL_CTRL` 需要拦截（计数器子集场景）

这在 `vmx_recalc_pmu_msr_intercepts()` 中配置：

```c
bool intercept = !has_mediated_pmu;
// 对于客户范围内的每个 GP 计数器：设置 intercept = !has_mediated_pmu
// 对于超出客户范围的计数器：始终拦截
// 对于 GLOBAL_STATUS/CTRL/OVF_CTRL：在计数器子集时拦截
```

### 2. 计数器精确度

**模拟 PMU**：计数是*近似的*。perf 子系统使用采样：它以负周期值编程硬件计数器并等待溢出。在溢出发生到 KVM 处理事件之间，额外的计数会累积（"滑移"）。此外：
- KVM 模拟指令的 `emulated_counter` 路径增加了不精确性
- 读取计数器需要 `perf_event_pause()` 来获取准确快照
- 宿主和客户 perf 事件之间的上下文切换可能在边界处丢失或重复计数

**中介 PMU**：计数是*硬件精确的*。客户机的计数器在真实硬件中运行，使用客户机精确的事件选择器配置。VM-Exit 时通过 `rdpmc()` 读回的计数器值反映了真实的硬件计数。唯一的近似来自 KVM 模拟指令——这些指令通过软件递增计数器。

### 3. 宿主-客户 PMU 共存

**模拟 PMU**：宿主和客户的 PMU 使用通过 perf 的事件调度自然共存。`exclude_host = 1` 属性确保客户事件仅在客户执行期间计数。多个客户机和宿主机可以通过 perf 管理的时分复用共享 PMU 硬件。但这种灵活性的代价是交叉映射问题和事件调度开销。

**中介 PMU**：客户机在执行期间拥有**PMU 独占访问权**。`perf_load_guest_context()` 通知宿主 perf 子系统完全让出 PMU。当客户机运行时，宿主 perf 事件无法使用 PMU。这意味着：
- 在中介 PMU 激活时，无法通过 `perf_event` 从宿主侧对客户机代码进行性能分析
- 宿主在 VM-Exit 时通过 `perf_put_guest_context()` 恢复 PMU
- 权衡是用独占性换取简洁性和性能

### 4. 事件过滤执行

**模拟 PMU**：通过在过滤器拒绝事件时*不编程* perf 事件来执行事件过滤。`reprogram_counter()` 检查 `pmc_is_event_allowed()`，如果事件被拒绝则跳过创建 perf 事件。

**中介 PMU**：由于计数器 MSR 访问为直通，事件过滤的工作方式不同。过滤通过操纵**硬件可见的使能位**来执行。对于通用计数器，事件选择器中的 `ENABLE` 位在 `eventsel_hw` 中被掩码。对于固定计数器，`fixed_ctr_ctrl_hw` 中的相应位被掩码。客户机看到自己的值，但硬件运行的是过滤后的版本：

```c
kvm_mediated_pmu_refresh_event_filter():
  if (pmc_is_gp(pmc)) {
      pmc->eventsel_hw &= ~ENABLE;
      if (allowed)
          pmc->eventsel_hw |= pmc->eventsel & ENABLE;
  } else {
      pmu->fixed_ctr_ctrl_hw &= ~mask;
      if (allowed)
          pmu->fixed_ctr_ctrl_hw |= pmu->fixed_ctr_ctrl & mask;
  }
```

### 5. 硬件要求

**模拟 PMU**：适用于任何 Intel PMU 版本 >= 1（或 AMD 等效版本）。由于 perf 抽象了硬件细节，硬件要求最低。

**中介 PMU**：需要特定的硬件特性（`intel_pmu_is_mediated_pmu_supported()`）：
- **PMU 架构版本 >= 4**：需要 `MSR_CORE_PERF_GLOBAL_STATUS_SET`，允许 KVM 精确加载客户机的溢出状态位
- **全宽度写入**（`PERF_CAP_FW_WRITES`）：需要通过 `MSR_IA32_PMCx`（而非 `MSR_IA32_PERFCTRx`）写入完整的 48 位计数器值，避免符号扩展问题
- **VMCS PERF_GLOBAL_CTRL 字段**：必须支持 `VM_ENTRY_LOAD_IA32_PERF_GLOBAL_CTRL`，用于 VM 转换时的原子使能/禁用

### 6. 上下文切换成本

**模拟 PMU**：VM-Entry/Exit 时上下文切换开销最小，因为 perf 事件是持久的宿主对象。事件仅在客户执行期间计数（`exclude_host = 1`）。但交叉映射检查和 perf 事件调度会增加延迟。

**中介 PMU**：上下文切换需要显式保存/恢复所有 PMU MSR。在 `kvm_mediated_pmu_load()` 时：写入 N 个 GP 计数器 + N 个 GP 选择器 + N 个固定计数器 + FIXED_CTR_CTRL + GLOBAL_STATUS。在 `kvm_mediated_pmu_put()` 时：读回 N 个 GP 计数器 + N 个固定计数器 + GLOBAL_STATUS + 清除硬件状态。对于 8 个 GP + 4 个固定计数器，这意味着每个 VM-Entry/Exit 周期约 30 次 MSR 读写。无论客户机是否正在使用 PMU，都要支付这一固定成本。

### 7. 嵌套虚拟化

**模拟 PMU**：与嵌套虚拟化自然兼容，因为 PMU 完全通过 perf 进行软件模拟。

**中介 PMU**：与嵌套虚拟化（L1 hypervisor 内运行的 L2 客户机）的交互需要谨慎处理。`vmx/mediated_pmu_freeze_in_smm` 分支名暗示了对 SMM（系统管理模式）交互等边缘情况的持续开发工作，在这些场景中 PMU 状态必须被冻结。

## 使用场景建议

| 场景 | 建议 |
|------|------|
| 生产环境 VM 性能分析，最小开销 | 中介 PMU |
| 从宿主侧分析客户机工作负载 | 模拟 PMU |
| 宿主与客户混合使用 PMU | 模拟 PMU |
| 客户机内运行 `perf` 工具进行自我分析 | 中介 PMU |
| 硬件不支持 PMU v4 / 全宽度写入 | 模拟 PMU（唯一选择） |
| 客户机内最大计数精度 | 中介 PMU |
| 计数器子集（客户机计数器少于宿主硬件） | 均可（中介模式为缺失计数器添加拦截） |

## 实现概要

```
                    模拟 PMU                              中介 PMU
                    ========                              ========

MSR 写入:      VM-Exit → KVM 拦截                  直接硬件写入（无退出）
               → reprogram_counter()
               → perf_event_create_kernel_counter()

计数器读取:    VM-Exit → KVM 拦截                  直接 RDPMC（无退出）
               → perf_event_pause() + emulated_counter

计数方式:      宿主上下文中的 perf_event            客户上下文中的硬件计数器
               (exclude_host=1)                    (VM-Entry 时通过 MSR 写入加载)

溢出处理:      perf 回调 → kvm_perf_overflow()     硬件 PMI → kvm_handle_guest_mediated_pmi()
               → KVM_REQ_PMI                       → KVM_REQ_PMI

启用/禁用:     perf_event_enable/disable            VMCS GUEST_IA32_PERF_GLOBAL_CTRL
                                                    (VM-Entry/Exit 时原子操作)

宿主共存:      通过 perf 调度器复用                 独占 (perf_load/put_guest_context)

关键代码:      pmu.c: reprogram_counter()          pmu.c: kvm_mediated_pmu_load/put()
               pmu.c: pmc_reprogram_counter()      pmu_intel.c: intel_mediated_pmu_load/put()
               pmu.c: kvm_perf_overflow()          vmx.c: vmx_recalc_pmu_msr_intercepts()
```

## 与硬件虚拟化发展轨迹的关系

中介 PMU 遵循与其他硬件虚拟化优化相同的模式（参见 [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md)）：

```
EPT      → 消除内存 VM-Exit        (影子页表 → 硬件遍历)
APICv    → 消除中断 VM-Exit        (软件注入 → posted interrupts)
VT-d     → 消除 I/O VM-Exit       (设备模拟 → 设备直通)
中介 PMU → 消除 PMU VM-Exit       (perf 模拟 → 直接硬件访问)
```

每一代技术都从另一个数据通路中移除了 hypervisor，逐渐趋近裸金属性能。中介 PMU 是这一发展轨迹向性能监控子系统的自然延伸。

## 另请参阅

- [kvm-pmu-virtualization](../entities/kvm-pmu-virtualization.md) -- KVM PMU 虚拟化：数据结构、MSR 处理、关键函数
- [kvm-cpu-virtualization](../entities/kvm-cpu-virtualization.md) -- VCPU 生命周期和 VM-Entry/Exit 机制
- [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md) -- PMI 传递和 APIC 模拟
- [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md) -- 渐进式 VM-Exit 消除
- [kvm-performance-tuning](../entities/kvm-performance-tuning.md) -- KVM 通用性能优化
- [cmp-emulated-vs-mediated-pmu](cmp-emulated-vs-mediated-pmu.md) -- English version
