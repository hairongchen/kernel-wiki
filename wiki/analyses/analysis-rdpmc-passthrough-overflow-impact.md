---
type: analysis
created: 2026-04-18
updated: 2026-04-18
sources: [qemu-kvm-source-code-and-application]
tags: [rdpmc, pmu, pmc, overflow, kvm, svm, vmx, emulated-pmu, mediated-pmu, passthrough]
---

# RDPMC 不拦截对 Guest 溢出处理的影响

本分析探讨在 KVM guest 中，如果 RDPMC 指令不被拦截（直接在硬件上执行），会如何影响 guest 内核的 PMC 溢出处理流程。核心结论：**影响取决于 PMU 虚拟化模式**——模拟模式下溢出处理被损坏，中介模式下则是正确且预期的设计。

## 1. 背景：RDPMC 在溢出处理中的角色

Guest 内核的 PMU NMI handler（无论 Intel 还是 AMD）在处理计数器溢出时，需要读取当前计数器值来：

1. **确认溢出**：检查 bit47 是否翻转（负值变正值）
2. **计算 delta**：`delta = current_raw - prev_count`，用于更新事件的总计数
3. **重装计数器**：基于 delta 计算新的 `-sample_period` 并写回

这个读取操作通过 `x86_perf_event_update()` 完成，内部使用 `rdpmcl()` 指令：

```c
x86_perf_event_update(event):
    prev = local64_read(&hwc->prev_count)    // 上次写入的值
    now  = rdpmcl(hwc->idx)                   // RDPMC 读取当前值
    delta = (now - prev) & counter_mask
    local64_add(delta, &event->count)         // 累计到事件总计数
    return now                                // 返回值用于 bit47 检查
```

**关键问题**：`rdpmcl()` 读到的值是否是 guest 期望的逻辑计数器值？

## 2. 模拟 PMU + RDPMC 不拦截 = 溢出处理损坏

### 2.1 根本原因：硬件计数器属于 host

在模拟（Emulated）PMU 模式下（AMD KVM 当前唯一模式），**硬件计数器由 host perf 子系统拥有和管理**，guest 的逻辑计数器值由 KVM 软件维护：

```
                    Guest 视角的计数器          硬件实际计数器
                    ══════════════════          ══════════════════

写入时:
  Guest WRMSR PerfCtr[n] = -1000        KVM → host perf → 硬件写 -5000
  (VM-Exit, KVM 记录 pmc->counter)      (perf 根据自己的 sample_period 编程)

运行后:
  Guest 认为计数器 ≈ -800               硬件实际 ≈ -3200
  (逻辑值 = pmc->counter +              (host perf_event 的计数进度，
   emulated_counter +                    与 guest 的 period 无关)
   perf_event_read_value)

RDPMC 拦截时:
  KVM 返回 -800                         （正确：pmc_read_counter() 计算逻辑值）

RDPMC 不拦截时:
  Guest 直接读到 -3200                  （错误！与 guest 期望值毫无关系）
```

### 2.2 Guest 溢出处理的具体损坏

以 AMD Zen 4+ `amd_pmu_v2_handle_irq()` 为例：

```
Guest 溢出处理流程：

① RDMSR GlobalStatus → VM-Exit → KVM 返回软件 global_status
   结果：正确 ✓（GlobalStatus 由 KVM 维护，不依赖 RDPMC）

② WRMSR GlobalCtl=0  → VM-Exit → KVM 更新 global_ctrl
   结果：正确 ✓（不涉及 RDPMC）

③ x86_perf_event_update(event):
   prev = hwc->prev_count               // Guest 上次 WRMSR 的值，如 -1000
   now  = rdpmcl(idx)                    // ← 不拦截！读到 host 值，如 -3200
   delta = (-3200 - (-1000)) & mask
         = -2200 & mask                  // 完全错误的 delta！
   ✗ 错误

④ 溢出判定：检查 now 的 bit47
   if !(now & (1ULL << 47)):             // 检查 host 计数器的 bit47
       → 可能误判溢出（host 值恰好 bit47=0）
       → 也可能漏判溢出（host 值 bit47=1 但 guest 已溢出）
   ✗ 不可靠

⑤ perf_event_overflow() → 基于错误的 delta 生成采样记录
   ✗ 采样数据的计数值损坏（IP/callchain 仍正确，来自 NMI regs）

⑥ 计数器重装：Guest WRMSR PerfCtr[n] = -(new_period)
   → VM-Exit → KVM reprogram
   → new_period 基于错误的 delta 调整 → 采样频率漂移
   ✗ 后续采样间隔错误
```

### 2.3 错误后果汇总

| 影响 | 严重度 | 说明 |
|------|--------|------|
| 溢出误判 | 严重 | bit47 检查基于 host 计数器状态，可能 false positive/negative |
| 事件计数错误 | 严重 | delta 基于不相关的两个值（guest prev vs host raw），累计值无意义 |
| 采样频率漂移 | 中等 | 重装的 sample_period 基于错误的 delta 调整，采样越来越不均匀 |
| perf stat 数据损坏 | 严重 | `perf stat` 报告的事件总数完全不可信 |
| perf record 部分损坏 | 中等 | 采样点的 IP/callchain 正确（来自 NMI regs），但关联的 period 值错误 |

### 2.4 为什么 KVM 必须拦截 RDPMC

KVM 在模拟模式下拦截 RDPMC 后的处理路径：

```c
// KVM 的 RDPMC 拦截处理
kvm_pmu_rdpmc(vcpu, ecx, &data):
  pmc = rdpmc_ecx_to_pmc(ecx)
  data = pmc_read_counter(pmc)          // 返回 guest 逻辑值

pmc_read_counter(pmc):                  // pmu.h
  // 模拟模式：组合三个来源
  counter = pmc->counter                // KVM 维护的基准值
          + pmc->emulated_counter       // KVM 模拟指令累计
  if (pmc->perf_event && !pmc->is_paused)
      counter += perf_event_read_value(perf_event)  // host perf 的实际计数
  return counter & pmc_bitmask(pmc)     // 正确的 guest 逻辑值
```

**拦截 RDPMC 使 KVM 能将三个独立的计数来源（基准值、模拟指令计数、perf 硬件计数）合并为 guest 期望看到的单一逻辑值。**

## 3. 中介 PMU + RDPMC 不拦截 = 正确工作（设计如此）

在中介（Mediated）PMU 模式下（当前仅 Intel PMU v4+ 支持），RDPMC 不拦截是**正常设计**，因为 KVM 在 VM-Entry 时将 guest 的计数器值直接加载到硬件：

```
VM-Entry: kvm_mediated_pmu_load()
  → kvm_pmu_load_guest_pmcs()
    for each counter:
        wrmsrq(PERFCTRx, pmc->counter)    // guest 值写入硬件
        wrmsrq(PERFEVTSELx, eventsel_hw)  // guest 事件选择器写入硬件
  → VMCS: GUEST_PERF_GLOBAL_CTRL = guest值  // 原子使能

[Guest 运行中]
  硬件计数器包含 guest 的值
  Guest RDPMC → 直接读硬件 → 读到 guest 自己的计数器值  ✓

[溢出发生]
  硬件：PerfCtr[n] bit47 翻转 → GlobalStatus[n]=1 → PMI
  → kvm_handle_guest_mediated_pmi() → KVM_REQ_PMI
  → Guest NMI handler:
    x86_perf_event_update():
      prev = hwc->prev_count              // guest 上次设置的值
      now  = rdpmcl(idx)                  // 硬件值 = guest 的实际计数
      delta = (now - prev) & mask         // 正确的 delta  ✓
    bit47 检查：基于 guest 自己的计数器值   // 正确  ✓

VM-Exit: kvm_mediated_pmu_put()
  → kvm_pmu_put_guest_pmcs()
    for each counter:
        pmc->counter = rdpmcl(idx)         // KVM 也用 RDPMC 保存 guest 值
```

### 3.1 中介模式下 RDPMC 正确的根本原因

```
中介模式的不变量（invariant）：

  在 guest 执行期间：
    硬件 PerfCtr[n] ≡ guest 的逻辑计数器值

  因此：
    Guest RDPMC = 读硬件 = 读 guest 逻辑值 = 正确

  对比模拟模式：
    硬件 PerfCtr[n] ≡ host perf_event 的内部计数器（与 guest 无关）
    Guest RDPMC = 读硬件 ≠ guest 逻辑值 = 错误
```

## 4. AMD 未来 Mediated PMU 场景

如果未来 AMD 实现 mediated PMU（需要 VMCB 扩展，见 [analysis-amd-pmc-overflow](analysis-amd-pmc-overflow.md#6-未来amd-mediated-pmu-的可能性与差距)），RDPMC 不拦截同样可以正确工作：

```
假设的 AMD mediated 溢出处理：

VM-Entry:
  WRMSR PerfCtr[0..5] = guest 值
  WRMSR PerfEvtSel[0..5] = guest 值
  VMCB 原子: PerfCntrGlobalCtl = guest 值（需未来硬件支持）

Guest 运行:
  RDPMC → 硬件值 = guest 值 → 正确                      ✓
  RDMSR GlobalStatus → 硬件值 → 正确                     ✓

溢出处理 (amd_pmu_v2_handle_irq):
  ① RDMSR GlobalStatus → 硬件（正确）                    ✓
  ② WRMSR GlobalCtl=0 → 硬件冻结（正确）                 ✓
  ③ RDPMC → 硬件 = guest 值（正确）                      ✓
  ④ x86_perf_event_update() delta 正确                   ✓
  ⑤ bit47 检查正确                                       ✓
  ⑥ 计数器重装正确                                       ✓
```

## 5. 总结

### 5.1 决策矩阵

```
┌──────────────────┬──────────────────┬─────────────────────────────────────┐
│ PMU 模式         │ RDPMC 拦截?      │ Guest 溢出处理                      │
├──────────────────┼──────────────────┼─────────────────────────────────────┤
│ 模拟 (Emulated)  │ 拦截 ✓           │ 正确：KVM pmc_read_counter() 返回   │
│                  │                  │ 合成的 guest 逻辑值                  │
├──────────────────┼──────────────────┼─────────────────────────────────────┤
│ 模拟 (Emulated)  │ 不拦截 ✗         │ 损坏：读到 host perf 的硬件计数器，  │
│                  │                  │ delta/bit47/重装全部错误             │
├──────────────────┼──────────────────┼─────────────────────────────────────┤
│ 中介 (Mediated)  │ 不拦截 ✓（设计）  │ 正确：硬件含 guest 值，RDPMC 读到   │
│                  │                  │ guest 自己的计数器                   │
├──────────────────┼──────────────────┼─────────────────────────────────────┤
│ 中介 (Mediated)  │ 拦截 △           │ 正确但有不必要的 VM-Exit 开销       │
└──────────────────┴──────────────────┴─────────────────────────────────────┘
```

### 5.2 根本原则

```
RDPMC 是否可以不拦截？取决于一个不变量：

  硬件 PerfCtr[n] ≡ guest 的逻辑计数器值？

  模拟模式：否（硬件属于 host perf）→ 必须拦截
  中介模式：是（硬件加载了 guest 值）→ 可以不拦截

这个不变量同时决定了：
  - RDPMC 读取是否正确
  - RDMSR PerfCtr[n] 是否正确（同理）
  - 硬件溢出检测（bit47 翻转）是否对 guest 有意义
  - 硬件 PMI 是否代表 guest 的溢出事件
```

### 5.3 当前 AMD 状态

AMD KVM 使用模拟模式，RDPMC **必须被拦截**。SVM 通过 VMCB 的 intercept 位控制 RDPMC 拦截。如果错误地清除此位，guest 的 `amd_pmu_v2_handle_irq()` 中 `x86_perf_event_update()` 读到的 delta 无意义，导致溢出检测错误、事件计数损坏、采样精度崩溃。

## See also

- [analysis-amd-pmc-overflow](analysis-amd-pmc-overflow.md) — AMD PMC 溢出处理机制全链路，含 Zen 4+ PerfMonV2 和 mediated PMU 差距分析
- [kvm-pmu-virtualization](../entities/kvm-pmu-virtualization.md) — KVM PMU 虚拟化：pmc_read_counter() 路径分岔、MSR 拦截配置
- [cmp-emulated-vs-mediated-pmu](../comparisons/cmp-emulated-vs-mediated-pmu.md) — 模拟 PMU vs 中介 PMU 对比：计数器精度、MSR 直通、溢出处理差异
- [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md) — 硬件虚拟化演进与 VM-Exit 消除轨迹
