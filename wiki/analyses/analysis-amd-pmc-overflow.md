---
type: analysis
created: 2026-04-18
updated: 2026-04-18
sources: [qemu-kvm-source-code-and-application]
tags: [amd, pmu, pmc, overflow, nmi, perf, kvm, svm, zen, perfmonv2, zen4]
---

# AMD PMC 溢出处理机制

本分析详细说明 AMD Performance Monitoring Counter（PMC）溢出在 Linux 内核中的处理流程，涵盖硬件层、host perf 驱动层和 KVM 虚拟化层。重点分析 Zen 4+（PerfMonV2）在 Host 和 Guest VM 中的溢出处理全链路，并与传统 AMD 和 Intel 方案进行对比。

## 目录

1. [AMD PMU 架构与 Intel 的关键差异](#1-amd-pmu-架构与-intel-的关键差异)
2. [传统 AMD 溢出检测机制（Zen 3 及更早）](#2-传统-amd-溢出检测机制zen-3-及更早)
3. [Zen 4+ PerfMonV2 硬件基础](#3-zen-4-perfmonv2-硬件基础)
4. [Host 端：perf 驱动的 PerfMonV2 溢出处理](#4-host-端perf-驱动的-perfmonv2-溢出处理)
5. [Guest VM 端：KVM SVM 的 PMU 溢出处理](#5-guest-vm-端kvm-svm-的-pmu-溢出处理)
6. [未来：AMD Mediated PMU 的可能性与差距](#6-未来amd-mediated-pmu-的可能性与差距)
7. [总结](#7-总结)

## 1. AMD PMU 架构与 Intel 的关键差异

| 特性 | Intel | AMD（Zen 3-） | AMD（Zen 4+） |
|------|-------|--------------|---------------|
| 全局溢出状态寄存器 | `IA32_PERF_GLOBAL_STATUS` (PMU v2+) | **无** | `PerfCntrGlobalStatus` |
| 全局溢出清除寄存器 | `IA32_PERF_GLOBAL_OVF_CTRL` | **无** | `PerfCntrGlobalStatusClr` |
| 全局溢出设置寄存器 | `IA32_PERF_GLOBAL_STATUS_SET` (v4+) | **无** | `PerfCntrGlobalStatusSet` |
| 全局使能/禁用 | `IA32_PERF_GLOBAL_CTRL` | **无** | `PerfCntrGlobalCtl` |
| 溢出源识别 | 读 `GLOBAL_STATUS` 一次性获取 | 逐计数器轮询 bit47 | 读 `GlobalStatus` 一次性获取 |
| 溢出期间冻结 | `FREEZE_PERFMON_ON_PMI` 或软件冻结 | **无法冻结** | `GlobalCtl=0` 软件冻结 |
| PMI 投递 | LVTPC，可配置为任意向量 | LVTPC，传统仅 NMI | LVTPC，NMI |
| GP 计数器数 | 4-8（架构相关） | 4（传统）/ 6（Zen 2+） | 6 |
| 固定计数器 | 3（v2+） | **无** | 引入（有限） |
| VMCS/VMCB 原子加载 GlobalCtl | 支持 | N/A | **不支持** |

## 2. 传统 AMD 溢出检测机制（Zen 3 及更早）

### 2.1 硬件触发流程

```
硬件溢出触发：

1. 软件将计数器 PerfCtr[n] 预设为负值（如 -sample_period）
   例如：48 位计数器写入 0xFFFF_FFFF_FF00（= -256 的 48 位补码表示）

2. 计数器递增，从 0xFFFF_FFFF_FFFF 翻转到 0x0000_0000_0000
   → 最高位（bit 47）从 1 变为 0

3. 若 PerfEvtSel[n].INT (bit 20) = 1
   → 硬件通过 LVTPC 投递 NMI

4. NMI 处理程序被触发
```

### 2.2 Host perf 驱动处理

内核的传统 AMD PMU 驱动位于 `arch/x86/events/amd/core.c`：

```
NMI 到达
  → nmi_handler()
    → perf_event_nmi_handler()
      → x86_pmu.handle_irq = amd_pmu_handle_irq()

amd_pmu_handle_irq():
  handled = 0
  for idx in 0..x86_pmu.num_counters:
      if idx not in cpuc->active_mask:
          continue
      event = cpuc->events[idx]
      val = x86_perf_event_update(event)

      // 溢出检测：利用有符号数判断
      // 计数器预设为负值，溢出后变为正值（高位翻转为 0）
      if val & (1ULL << (x86_pmu.cntval_bits - 1)):
          continue    // 高位仍为 1 → 值仍为负 → 未溢出

      handled++
      perf_event_overflow(event, &data, regs)
  return handled
```

**核心原理**：

```
预设阶段：
  counter = -period = 0xFFFF_FFFF_FF00  (48位补码, bit47=1, 负值)

计数过程：
  0xFFFF_FFFF_FF00 → ... → 0xFFFF_FFFF_FFFF → 0x0000_0000_0000
                                                 ↑ 溢出点

溢出检测：
  读取 counter 值，检查 bit 47：
    bit47 = 1 → 值仍为负 → 未溢出
    bit47 = 0 → 值已变正 → 溢出发生
```

### 2.3 传统 AMD NMI 处理的固有问题

| 问题 | 说明 |
|------|------|
| NMI 源歧义 | NMI 可能来自 PMC、watchdog、硬件错误，`amd_pmu_handle_irq()` 必须猜测 |
| 无法冻结计数器 | 处理 NMI 期间计数器继续运行，产生额外计数（skid） |
| 全遍历开销 | 即使只有 1 个计数器溢出，仍需遍历全部 6 个计数器 |
| 多计数器同时溢出 | 需逐一检查，不能一次性确定所有溢出源 |

## 3. Zen 4+ PerfMonV2 硬件基础

### 3.1 CPUID 检测

Zen 4 引入 `CPUID Fn 8000_0022` 叶，通告 PerfMonV2 能力：

```
CPUID Fn 8000_0022:
  EAX bit 0: PerfMonV2           — 支持全局 PMU 控制寄存器
  EBX[3:0]:  NumCorePerfCtr      — 核心 GP 计数器数（Zen 4: 6）
  EBX[9:4]:  NumDfPerfCtr        — Data Fabric 计数器数
  EBX[13:10]: NumUmcPerfCtr      — 统一内存控制器计数器数
```

内核通过 `X86_FEATURE_PERFMON_V2`（`boot_cpu_has()` 检测）决定是否启用 V2 路径。

### 3.2 新增 MSR

| MSR 地址 | 名称 | 读/写 | 功能 |
|----------|------|-------|------|
| `0xC0000300` | `PerfCntrGlobalStatus` | 只读 | 溢出状态位，每个 GP/L3 计数器一位 |
| `0xC0000301` | `PerfCntrGlobalStatusClr` | 只写 | 写 1 清除对应溢出位 |
| `0xC0000302` | `PerfCntrGlobalStatusSet` | 只写 | 写 1 设置对应溢出位（测试/状态恢复） |
| `0xC0000303` | `PerfCntrGlobalCtl` | 读/写 | 全局使能/禁用，每个计数器一位 |

### 3.3 GlobalCtl 与 PerfEvtSel.EN 的关系

`PerfCntrGlobalCtl[n]` 与 `PerfEvtSel[n].EN`（bit 22）是 **AND 关系**：

```
计数器[n] 运行条件 = PerfCntrGlobalCtl[n] AND PerfEvtSel[n].EN

效果：
  GlobalCtl[n]=0, EN=1  → 计数器停止（全局禁用）
  GlobalCtl[n]=1, EN=0  → 计数器停止（局部禁用）
  GlobalCtl[n]=1, EN=1  → 计数器运行
  GlobalCtl[n]=0, EN=0  → 计数器停止
```

这使得单次 `WRMSR PerfCntrGlobalCtl = 0` 即可原子性地冻结所有计数器，无需逐一修改 `PerfEvtSel[n]`。

## 4. Host 端：perf 驱动的 PerfMonV2 溢出处理

### 4.1 初始化

```
amd_pmu_v2_init():                    // arch/x86/events/amd/core.c
  if (!boot_cpu_has(X86_FEATURE_PERFMON_V2))
      return -ENODEV

  // 替换关键函数指针
  x86_pmu.handle_irq  = amd_pmu_v2_handle_irq    // NMI handler
  x86_pmu.enable       = amd_pmu_v2_enable_event  // 计数器使能
  x86_pmu.disable      = amd_pmu_v2_disable_event // 计数器禁用
  x86_pmu.enable_all   = amd_pmu_v2_enable_all    // 全局使能
  x86_pmu.disable_all  = amd_pmu_v2_disable_all   // 全局禁用

  // 从 CPUID 读取计数器数量
  num_core_pmc = CPUID_8000_0022_EBX[3:0]        // Zen 4: 6
  x86_pmu.num_counters = num_core_pmc
```

### 4.2 计数器使能/禁用

PerfMonV2 引入双层使能机制，`enable`/`disable` 同时操作 `PerfEvtSel` 和 `GlobalCtl`：

```
amd_pmu_v2_enable_event(event):
  // ① 写 PerfEvtSel[n]：设置事件选择器 + EN 位
  __x86_pmu_enable_event(hwc, ARCH_PERFMON_EVENTSEL_ENABLE)
  // ② 通过 GlobalCtl 使能该计数器位
  amd_pmu_set_global_ctl(BIT_ULL(hwc->idx))

amd_pmu_v2_disable_event(event):
  // 仅通过 GlobalCtl 禁用（不修改 PerfEvtSel，保留事件配置）
  amd_pmu_clr_global_ctl(BIT_ULL(hwc->idx))

amd_pmu_v2_enable_all(added):
  amd_pmu_set_global_ctl(active_mask)    // 一次 WRMSR 使能所有活跃计数器

amd_pmu_v2_disable_all():
  amd_pmu_clr_global_ctl(active_mask)    // 一次 WRMSR 禁用所有
```

### 4.3 溢出处理：`amd_pmu_v2_handle_irq()`

这是 Zen 4+ 的核心改进——**取代逐计数器轮询**：

```
amd_pmu_v2_handle_irq(regs):
  struct cpu_hw_events *cpuc = this_cpu_ptr(&cpu_hw_events)

  // ① 读取全局溢出状态——单次 RDMSR
  status = rdmsrq(MSR_AMD64_PERF_CNTR_GLOBAL_STATUS)
  if (!status)
      return 0                    // 非 PMC 引起的 NMI → 明确判定

  // ② 全局冻结所有计数器——单次 WRMSR
  //    防止处理期间计数器继续运行（消除 skid）
  amd_pmu_v2_disable_all()        // GlobalCtl = 0

  // ③ 仅遍历 status 中置位的计数器（精确定位溢出源）
  status &= active_mask           // 过滤非活跃计数器
  for_each_set_bit(idx, &status, x86_pmu.num_counters):
      event = cpuc->events[idx]
      if (!event)
          continue

      hwc = &event->hw
      val = x86_perf_event_update(event)

      // 双重确认：GlobalStatus 置位 + bit47 翻转
      if !(val & (1ULL << (x86_pmu.cntval_bits - 1))):
          handled++
          perf_event_overflow(event, &data, regs)

  // ④ 清除已处理的溢出位——单次 WRMSR
  wrmsrq(MSR_AMD64_PERF_CNTR_GLOBAL_STATUS_CLR, status)

  // ⑤ 重新使能计数器——单次 WRMSR
  amd_pmu_v2_enable_all(0)        // GlobalCtl = active_mask

  return handled
```

### 4.4 传统 vs PerfMonV2 溢出处理对比

```
┌─────────────────────────────────┬──────────────────────────────────────┐
│       Zen 3-（传统）            │         Zen 4+（PerfMonV2）          │
├─────────────────────────────────┼──────────────────────────────────────┤
│ NMI → 遍历所有 6 个计数器       │ NMI → RDMSR GlobalStatus 一次       │
│ 每个计数器：RDMSR + bit47 检查   │ 仅遍历 status 中置位的计数器         │
│ 无法冻结：处理期间计数器继续跑    │ GlobalCtl=0 冻结 → 消除 skid        │
│ 多计数器溢出：可能漏检           │ 精确：所有溢出位一次性获取           │
│ NMI 源歧义：需猜测是否 PMC       │ status==0 → 明确非 PMC NMI          │
│ ~6 次 RDMSR + 6 次 bit47 检查   │ 1 RDMSR + 2 WRMSR（冻结+恢复）     │
│ 总 MSR 操作：~6-12 次           │ 总 MSR 操作：~3-5 次               │
└─────────────────────────────────┴──────────────────────────────────────┘
```

## 5. Guest VM 端：KVM SVM 的 PMU 溢出处理

### 5.1 当前状态：仍为模拟（Emulated）模式

即使在 Zen 4+ 硬件上，KVM AMD 的 PMU 虚拟化**仍采用模拟模式**（通过 host `perf_event`）。AMD 不支持 mediated PMU。所有 Guest PMU MSR 访问触发 VM-Exit。

### 5.2 Guest PMC 溢出的完整路径

```
Guest 计数器溢出（模拟模式）：

t0: Guest 写 PerfEvtSel[n]（配置事件 + 使能计数）
    → VM-Exit（MSR 写入拦截）
    → KVM: kvm_pmu_set_msr()
      → reprogram_counter()
        → perf_event_create_kernel_counter()
          attr.exclude_host = 1
          attr.sample_period = (-counter) & bitmask

t1: Host perf_event 在 guest 上下文中到期（硬件计数器溢出）
    → Host NMI
    → Host perf handler（amd_pmu_v2_handle_irq 或传统 handler）
    → perf 回调: kvm_perf_overflow()
      → __kvm_perf_overflow(pmc, in_pmi=true)
        → pmu->global_status |= BIT_ULL(pmc->idx)    // KVM 内部软件位图
        → kvm_make_request(KVM_REQ_PMI, vcpu)

t2: KVM_RUN 入口检查 pending 请求
    → kvm_pmu_deliver_pmi(vcpu)
      → kvm_apic_local_deliver(APIC_LVTPC)
      → 注入 NMI 到 guest

t3: Guest NMI handler 执行
    → Guest 内核 amd_pmu_v2_handle_irq()（若 guest 检测到 PerfMonV2）
    → Guest RDMSR PerfCntrGlobalStatus  → VM-Exit → KVM 返回 pmu->global_status
    → Guest WRMSR PerfCntrGlobalCtl=0   → VM-Exit → KVM 更新 pmu->global_ctrl
    → Guest 读取溢出计数器并重装
      → Guest RDMSR PerfCtr[n]          → VM-Exit → KVM 返回 pmc_read_counter()
      → Guest WRMSR PerfCtr[n]          → VM-Exit → KVM pmc_write_counter()
    → Guest WRMSR GlobalStatusClr       → VM-Exit → KVM 清除 global_status 位
    → Guest WRMSR PerfCntrGlobalCtl=mask → VM-Exit → KVM 恢复 global_ctrl
```

### 5.3 Guest PerfMonV2 MSR 的 KVM 模拟

KVM 在 `arch/x86/kvm/svm/pmu.c` 中模拟 Zen 4+ 的全局 PMU MSR：

| Guest MSR 操作 | VM-Exit? | KVM 处理 |
|---------------|----------|---------|
| RDMSR `PerfCntrGlobalStatus` | 是 | 返回 `pmu->global_status`（KVM 软件维护） |
| WRMSR `PerfCntrGlobalStatusClr` | 是 | `pmu->global_status &= ~val` |
| WRMSR `PerfCntrGlobalStatusSet` | 是 | `pmu->global_status \|= val` |
| RDMSR `PerfCntrGlobalCtl` | 是 | 返回 `pmu->global_ctrl` |
| WRMSR `PerfCntrGlobalCtl` | 是 | 更新 `pmu->global_ctrl`，重编程受影响的计数器 |
| RDMSR/WRMSR `PerfEvtSel[n]` | 是 | 更新 `pmc->eventsel`，请求计数器重编程 |
| RDMSR/WRMSR `PerfCtr[n]` | 是 | `pmc_read_counter()` / `pmc_write_counter()` |

**关键点**：Guest 看到完整的 PerfMonV2 行为（由 KVM 软件模拟），Guest 内核的 `amd_pmu_v2_handle_irq()` 正常工作。但每次 MSR 读/写都触发 VM-Exit，性能开销远高于 host 原生执行。

### 5.4 每次 Guest 溢出处理的 VM-Exit 开销

Guest 的 `amd_pmu_v2_handle_irq()` 在一次溢出处理中产生的 VM-Exit：

| 操作 | VM-Exit 次数 | 说明 |
|------|-------------|------|
| RDMSR GlobalStatus | 1 | 读取溢出位 |
| WRMSR GlobalCtl=0 | 1 | 冻结所有计数器 |
| RDMSR PerfCtr[n] (溢出的) | 1-6 | 读取溢出计数器值 |
| WRMSR PerfCtr[n] (重装) | 1-6 | 重装计数器初始值 |
| WRMSR GlobalStatusClr | 1 | 清除溢出位 |
| WRMSR GlobalCtl=mask | 1 | 恢复计数器使能 |
| **合计** | **6-16** | 单个计数器溢出：~6 次；多计数器：更多 |

以 Zen 4 VM-Exit roundtrip ~600-900 ns 计算，单次溢出处理开销约 **3.6-14.4 μs**。

### 5.5 KVM 内部的 global_status 抽象

KVM 统一使用 `pmu->global_status` 位图来跟踪溢出状态，**无论底层是 Intel 还是 AMD**。这一软件抽象使得 Intel 和 AMD 的上层溢出处理逻辑可以共享：

```
                KVM 通用层（pmu.c）
                ┌─────────────────────────────┐
                │  pmu->global_status         │  ← 统一的溢出位图
                │  kvm_perf_overflow()        │  ← 统一的溢出回调
                │  kvm_pmu_deliver_pmi()      │  ← 统一的 PMI 投递
                └──────────┬──────────────────┘
                           │
              ┌────────────┼────────────────┐
              │                             │
    Intel (pmu_intel.c)           AMD (svm/pmu.c)
    ┌─────────────────┐           ┌─────────────────┐
    │ intel_pmu_ops   │           │ amd_pmu_ops     │
    │ MSR_P6_EVNTSEL0 │           │ MSR_K7_EVNTSEL0 │
    │ PERF_GLOBAL_*   │           │ PerfCntrGlobal* │
    │ mediated PMU ✓  │           │ mediated PMU ✗  │
    └─────────────────┘           └─────────────────┘
```

### 5.6 Guest 中传统 vs PerfMonV2 路径选择

Guest 内核根据 KVM 向 guest 暴露的 CPUID 来决定使用哪条路径：

- 若 KVM 向 guest 暴露 `CPUID 8000_0022.EAX[0]=1`（PerfMonV2），guest 使用 `amd_pmu_v2_handle_irq()`
- 否则 guest 回退到传统 `amd_pmu_handle_irq()`

两条路径在模拟模式下**性能差异不大**，因为瓶颈在 VM-Exit 开销而非 NMI handler 内部逻辑。但 PerfMonV2 路径为 guest 提供了更精确的溢出识别语义。

## 6. 未来：AMD Mediated PMU 的可能性与差距

### 6.1 Zen 4+ 已具备的硬件基础

PerfMonV2 的四个全局 MSR 为 mediated PMU 提供了必要条件：

| 能力 | 需求 | Zen 4+ 状态 |
|------|------|------------|
| 全局冻结/恢复计数器 | `PerfCntrGlobalCtl` | 已具备 |
| 精确保存溢出状态 | `PerfCntrGlobalStatus` (读) | 已具备 |
| 精确恢复溢出状态 | `PerfCntrGlobalStatusSet` (写) | 已具备 |
| 清除溢出状态 | `PerfCntrGlobalStatusClr` (写) | 已具备 |

### 6.2 仍缺少的关键 VMCB 支持

要实现 AMD mediated PMU，需要 VMCB 层面的原子保存/恢复支持。当前对比：

| 需求 | Intel VMCS | AMD VMCB（Zen 4） |
|------|-----------|-------------------|
| VM-Entry 原子加载 GlobalCtl | `VM_ENTRY_LOAD_IA32_PERF_GLOBAL_CTRL` 字段 | **不存在** |
| VM-Exit 原子保存 GlobalCtl | `VM_EXIT_SAVE_IA32_PERF_GLOBAL_CTRL` 字段 | **不存在** |
| GlobalStatus 自动保存/恢复 | VMCS guest-state 字段 | **不存在** |
| HOST_PERF_GLOBAL_CTRL | VMCS host-state 字段（VM-Exit 恢复 host 值） | **不存在** |
| PMU MSR bitmap 直通 | 完善（PerfEvtSel/PerfCtr/Fixed 可直通） | 部分（PerfEvtSel/PerfCtr 可通过 MSR bitmap 直通） |

### 6.3 无 VMCB 原子支持时的软件模拟困境

如果尝试纯软件实现 AMD mediated PMU（无 VMCB 原子支持），存在以下竞态窗口：

```
软件模拟 mediated PMU（假设路径）：

VM-Entry:
  ① WRMSR PerfCntrGlobalCtl = 0        // 冻结
  ② WRMSR PerfEvtSel[0..5] = guest值    // 加载 guest 事件选择器
  ③ WRMSR PerfCtr[0..5] = guest值       // 加载 guest 计数器值
  ④ WRMSR PerfCntrGlobalCtl = guest值   // 使能 guest 计数器
  ⑤ VMRUN                               // 进入 guest
     ↑ 窗口：步骤②-④之间约 6-12 次 WRMSR（~1-2 μs）
       期间 guest 事件选择器已加载但计数器值可能不一致

VM-Exit:
  ⑥ WRMSR PerfCntrGlobalCtl = 0        // 冻结
     ↑ 问题：VMRUN 退出到这里之间有窗口，计数器可能多计
  ⑦ RDMSR PerfCtr[0..5] → 保存          // 读回 guest 计数器
  ⑧ RDMSR PerfCntrGlobalStatus → 保存   // 读回溢出状态
  ⑨ 恢复 host PMU 状态
```

Intel 的 VMCS 原子字段消除了这些窗口——`VM_ENTRY_LOAD_IA32_PERF_GLOBAL_CTRL` 在 VMRESUME 微码中原子加载，`VM_EXIT_SAVE` 在 VMEXIT 微码中原子保存，无任何竞态。

### 6.4 实现 AMD Mediated PMU 的路线图

```
当前（Zen 4/5）                     未来可能
┌─────────────────────┐             ┌──────────────────────────────┐
│ 模拟 PMU（perf-based）│             │ Mediated PMU（直接硬件访问）   │
│                     │             │                              │
│ 所有 MSR → VM-Exit  │    需要     │ PerfEvtSel/PerfCtr → 直通     │
│ perf_event 代理计数  │  ──────→   │ 硬件直接计数                   │
│ 溢出通过 perf 回调   │  VMCB 扩展  │ 硬件 PMI 直接投递              │
│                     │             │                              │
│ ~6-16 VM-Exit/溢出  │             │ ~0-2 VM-Exit/溢出             │
└─────────────────────┘             └──────────────────────────────┘

所需 VMCB 扩展：
  1. VMCB guest-state 区增加 PerfCntrGlobalCtl 字段
  2. VMCB control 区增加 LOAD_PERF_GLOBAL_CTL 位
  3. VMCB control 区增加 SAVE_PERF_GLOBAL_CTL 位
  4. VMCB host-state 区增加 HOST_PERF_GLOBAL_CTL 字段
     （VM-Exit 时自动恢复 host GlobalCtl = 0，防止 guest 计数器泄漏）
```

## 7. 总结

### 7.1 Host 端全景

```
                        Host 原生溢出处理

硬件层：
  PerfCtr[n] 预设负值 → 递增 → bit47 翻转 → GlobalStatus[n]=1 → NMI via LVTPC

Kernel perf 驱动层 (arch/x86/events/amd/core.c):
  amd_pmu_v2_handle_irq()
  ├── RDMSR GlobalStatus              ← ① 一次读，精确定位溢出源
  ├── WRMSR GlobalCtl = 0             ← ② 一次写，全局冻结（消除 skid）
  ├── for_each_set_bit(status):       ← ③ 仅处理溢出计数器
  │     x86_perf_event_update()
  │     perf_event_overflow()          →  用户态采样/profiling
  ├── WRMSR GlobalStatusClr = status  ← ④ 一次写，清除溢出位
  └── WRMSR GlobalCtl = active_mask   ← ⑤ 一次写，恢复计数
```

### 7.2 Guest VM 端全景

```
                        Guest VM 溢出处理（模拟模式）

KVM 层 (arch/x86/kvm/pmu.c + svm/pmu.c):
  Host perf_event 到期
  → kvm_perf_overflow()
    → pmu->global_status |= BIT(idx)     // KVM 内部软件位图
    → KVM_REQ_PMI
  → kvm_pmu_deliver_pmi()
    → kvm_apic_local_deliver(LVTPC)
    → NMI 注入 guest

Guest Kernel (arch/x86/events/amd/core.c):
  amd_pmu_v2_handle_irq()                // 与 host 相同的代码路径
  ├── RDMSR GlobalStatus    → VM-Exit → KVM 返回软件 global_status
  ├── WRMSR GlobalCtl=0     → VM-Exit → KVM 更新 global_ctrl
  ├── 处理溢出计数器         → VM-Exit × N（读/写 PerfCtr）
  ├── WRMSR StatusClr       → VM-Exit → KVM 清除 global_status
  └── WRMSR GlobalCtl=mask  → VM-Exit → KVM 恢复 global_ctrl
      总计: ~6-16 次 VM-Exit / 每次溢出处理
```

### 7.3 三层对比

| 维度 | Host（Zen 4+） | Guest VM（Zen 4+ KVM） | Guest（假设未来 Mediated） |
|------|---------------|----------------------|--------------------------|
| 溢出检测 | 硬件 GlobalStatus | KVM 软件 global_status | 硬件 GlobalStatus |
| 冻结机制 | WRMSR GlobalCtl=0 | VM-Exit → KVM 软件更新 | VMCB 原子冻结（需硬件） |
| MSR 操作开销 | ~3-5 次 WRMSR/RDMSR | 同等次数 × VM-Exit 开销 | ~3-5 次原生 MSR |
| 每次溢出延迟 | ~0.5-1 μs | ~3.6-14.4 μs | ~0.5-1 μs（理论） |
| NMI 源识别 | GlobalStatus==0 → 非 PMC | N/A（KVM 直接注入） | 硬件原生 |
| 计数精度 | 硬件精确 | 近似（perf 采样 + skid） | 硬件精确 |

## 关键源文件

| 文件 | 内容 |
|------|------|
| `arch/x86/events/amd/core.c` | AMD host PMU 驱动：`amd_pmu_v2_handle_irq()`、`amd_pmu_v2_enable/disable_event()`、PerfMonV2 初始化 |
| `arch/x86/include/asm/perf_event.h` | MSR 定义：`MSR_AMD64_PERF_CNTR_GLOBAL_STATUS/CTL/CLR/SET` |
| `arch/x86/include/asm/cpufeatures.h` | `X86_FEATURE_PERFMON_V2` 特征位定义 |
| `arch/x86/kvm/svm/pmu.c` | AMD KVM PMU ops：`amd_pmu_ops`，PerfMonV2 MSR 模拟 |
| `arch/x86/kvm/pmu.c` | 通用 KVM PMU 逻辑：`kvm_perf_overflow()`、`reprogram_counter()` |
| `arch/x86/kvm/pmu.h` | `struct kvm_pmu_ops`、`kvm_pmc` 内联辅助函数 |

## See also

- [kvm-pmu-virtualization](../entities/kvm-pmu-virtualization.md) — KVM PMU 虚拟化数据结构与关键函数
- [cmp-emulated-vs-mediated-pmu](../comparisons/cmp-emulated-vs-mediated-pmu.md) — 模拟 PMU vs 中介 PMU 对比
- [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md) — PMI 通过 LVTPC 投递的中断虚拟化机制
- [analysis-epyc-lapic-timer](analysis-epyc-lapic-timer.md) — AMD EPYC 平台 LAPIC Timer 分析
- [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md) — 硬件虚拟化演进
