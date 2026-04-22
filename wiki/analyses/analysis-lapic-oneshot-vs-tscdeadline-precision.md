---
type: analysis
created: 2026-04-11
updated: 2026-04-11
sources: [understanding-the-linux-kernel, qemu-kvm-source-code-and-application, minimizing-vmexits-pv-ipi-passthrough-timer, bytedance-solution-vmexit]
tags: [lapic-timer, one-shot, tsc-deadline, precision, jitter, clockevents, hrtimer, timer-accuracy]
---

# LAPIC One-shot 模式 vs TSC-Deadline 模式精度差别研究

## 1. 问题背景

LAPIC Timer 是 x86 平台每个 CPU 核心内置的定时器，Linux 内核通过 `clockevents` 框架使用它作为核心时钟事件源。LAPIC Timer 有三种工作模式（Periodic / One-shot / TSC-Deadline），其中 **One-shot** 和 **TSC-Deadline** 均用于 tickless（`NO_HZ`）场景下按需触发中断。两者在定时精度上存在**根本性差异**，理解这一差异对内核定时子系统调优和虚拟化场景下的 timer 性能分析至关重要。

---

## 2. 硬件工作原理对比

### 2.1 One-shot 模式

One-shot 模式使用 LAPIC 内部的 **递减计数器**，由外部总线时钟（bus clock）驱动：

```
编程流程：
  1. 写 Divide Configuration Register (DCR / TDCR)
     → 设置分频比 (1, 2, 4, 8, 16, 32, 64, 128)
  2. 设置 LVT Timer Register: bit[17]=0 (one-shot), vector, mask
  3. 写 Initial Count Register (ICR)
     → 写入后立即开始递减

递减时钟源：
  计数频率 = bus_clock / divide_value

  例如：bus_clock = 100 MHz, divide = 1
        → 计数频率 = 100 MHz → 每 10 ns 递减一次

  ICR → CCR (Current Count Register):
  ┌────────┐    bus_clock / div    ┌────────┐
  │ ICR=N  │ ──────────────────→   │ CCR    │ = N, N-1, N-2, ..., 1, 0 → 触发中断
  └────────┘    每 tick 减 1        └────────┘
```

**精度受限因素：**

| 因素 | 说明 | 影响量级 |
|------|------|---------|
| 总线频率分辨率 | 计时单位 = 1/(bus_clock/div)，典型 10-100 ns | 最小定时粒度受限 |
| 分频器离散化 | 只有 1/2/4/8/16/32/64/128 八档 | 无法精确匹配任意时间间隔 |
| 总线频率不确定性 | bus clock 频率因平台而异且可能动态变化 | 引入校准误差 |
| 校准误差传播 | 内核启动时需通过 PIT/TSC 校准 bus clock 频率 | 典型 0.1-1% 系统性偏差 |
| ns→count 取整误差 | 将纳秒数转换为 tick 数时必须取整 | 每次编程引入 ±1 tick 量化误差 |

### 2.2 TSC-Deadline 模式

TSC-Deadline 模式使用 CPU 核心内部的 **TSC（Time Stamp Counter）** 作为时基，直接比较 TSC 与目标值：

```
编程流程：
  1. 设置 LVT Timer Register: bit[18]=1 (TSC-deadline mode)
  2. 写 IA32_TSC_DEADLINE MSR (MSR 0x6E0) = 目标 TSC 值
     → 当 TSC ≥ 目标值时，硬件自动触发中断

比较机制（硬件实现）：
  ┌──────────┐              ┌─────────────────┐
  │ TSC 计数器 │ ── ≥ ? ──→  │ TSC_DEADLINE MSR │
  └──────────┘    比较器      └─────────────────┘
       │                            │
       │          当 TSC ≥ deadline  │
       └─────── 触发 LVT Timer 中断 ←┘

  TSC 频率 = CPU 基础频率（Invariant TSC 下恒定）
  例如：TSC freq = 2.5 GHz → 分辨率 = 0.4 ns
```

**精度优势：**

| 因素 | 说明 | 影响量级 |
|------|------|---------|
| TSC 频率极高 | 通常等于 CPU 基频（2-5 GHz） | 分辨率 0.2-0.5 ns |
| 无分频器 | 直接使用 TSC 原始频率 | 无离散化损失 |
| 目标值绝对化 | 写入的是绝对 TSC 目标值 | 无累积误差 |
| Invariant TSC | 现代 CPU 的 TSC 不受 P-state/C-state 影响 | 无频率漂移 |
| 硬件直接比较 | CPU 微架构内部持续比较 TSC 与 deadline | 无软件轮询延迟 |

---

## 3. 精度差异的量化分析

### 3.1 时间分辨率对比

```
One-shot 模式：
  分辨率 = 1 / (bus_clock / divide_value)

  典型场景 (bus_clock = 100 MHz, div = 1):
    分辨率 = 10 ns → 能表达的最小定时单位

  典型场景 (bus_clock = 200 MHz, div = 1):
    分辨率 = 5 ns

  典型场景 (bus_clock = 100 MHz, div = 16):
    分辨率 = 160 ns

TSC-Deadline 模式：
  分辨率 = 1 / TSC_frequency

  AMD EPYC 7763 (Zen 3, 2.45 GHz base):
    分辨率 = 1/2.45G ≈ 0.408 ns

  Intel Xeon Platinum 8380 (Ice Lake, 2.3 GHz base):
    分辨率 = 1/2.3G ≈ 0.435 ns

  AMD EPYC 9654 (Zen 4, 2.4 GHz base):
    分辨率 = 1/2.4G ≈ 0.417 ns

精度比：TSC-Deadline 分辨率通常比 One-shot 高 10-50 倍
```

### 3.2 编程延迟与触发抖动

#### One-shot 模式的编程路径

```c
// arch/x86/kernel/apic/apic.c
static int lapic_next_event(unsigned long delta,
                            struct clock_event_device *evt)
{
    apic_write(APIC_TMICT, delta);  // MMIO 写入 Initial Count Register
    return 0;
}
```

**延迟分解：**

```
1. ns_to_count 转换：delta_ns × bus_freq / (NSEC_PER_SEC × div)
   → 取整误差: ±1 count = ±(1/bus_freq × div) 秒
   → 例如 bus=100MHz, div=1: ±10 ns 量化误差

2. MMIO 写入延迟：APIC_TMICT 位于 LAPIC MMIO 空间 (0xFEE00000+0x380)
   → 典型 ~20-50 ns（本地 APIC 寄存器访问）

3. 计数器启动延迟：写入 ICR 到计数器开始递减的延迟
   → 同步到总线时钟域: 0-1 个总线时钟周期

4. 到期触发延迟：CCR 从 1 减到 0 → 中断信号生成
   → 硬件内部处理: ~1-5 ns

总编程-到-触发抖动（one-shot）: ~20-100 ns（不含中断投递延迟）
```

#### TSC-Deadline 模式的编程路径

```c
// arch/x86/kernel/apic/apic.c
static int lapic_next_deadline(unsigned long delta,
                               struct clock_event_device *evt)
{
    u64 tsc = rdtsc();
    wrmsrl(MSR_IA32_TSC_DEADLINE, tsc + (((u64) delta) * TSC_DIVISOR));
    return 0;
}
```

**延迟分解：**

```
1. rdtsc 执行：读取当前 TSC 值
   → ~20-30 cycles（非序列化，可乱序执行）
   → 在 2.5 GHz CPU 上: ~8-12 ns

2. 目标值计算：tsc + delta × TSC_DIVISOR
   → 纯算术运算: ~1-2 ns

3. WRMSR 执行：写入 MSR_IA32_TSC_DEADLINE (0x6E0)
   → ~30-50 cycles（序列化操作）
   → 在 2.5 GHz CPU 上: ~12-20 ns

4. 硬件比较生效：MSR 写入到比较器开始工作的延迟
   → 几乎立即（同一时钟域内）: ~1-2 cycles

5. 触发精度：TSC ≥ deadline 的检测延迟
   → 取决于微架构实现，典型 1-几 个 cycle: ~0.4-2 ns

总编程-到-触发抖动（TSC-deadline）: ~5-30 ns（不含中断投递延迟）
```

### 3.3 量化对比总结

```
┌─────────────────────┬─────────────────────┬─────────────────────┐
│       指标           │    One-shot 模式     │  TSC-Deadline 模式   │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ 时基分辨率           │ 5-100 ns            │ 0.2-0.5 ns          │
│ (取决于 bus clock    │ (bus_clock/div)     │ (1/TSC_freq)        │
│  和分频比)           │                     │                     │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ 量化误差 (每次编程)  │ ±1 bus tick         │ ±1 TSC cycle        │
│                     │ = ±5-100 ns         │ = ±0.2-0.5 ns       │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ 编程操作延迟         │ ~20-50 ns (MMIO)    │ ~12-20 ns (WRMSR)   │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ 触发抖动 (Jitter)   │ ~20-100 ns          │ ~5-30 ns            │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ 时钟源稳定性         │ 受总线频率变化影响    │ Invariant TSC 恒定   │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ 校准需求             │ 需校准 bus clock     │ CPUID 可直接获取      │
│                     │ 频率 (可能有偏差)    │ TSC 频率 (精确)      │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ 长时间定时漂移       │ 可能（bus clock      │ 极小（Invariant TSC   │
│                     │  频率微小偏差累积）   │  晶振级稳定性）       │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ 典型精度等级         │ ~100 ns 级          │ ~1-10 ns 级          │
└─────────────────────┴─────────────────────┴─────────────────────┘
```

---

## 4. 深层原因分析

### 4.1 时钟域差异

One-shot 模式与 TSC-Deadline 模式的精度差异根本上源于 **时钟域的不同**：

```
                          CPU 芯片内部
  ┌─────────────────────────────────────────────────────┐
  │                                                     │
  │   Core 时钟域                  Bus/Uncore 时钟域     │
  │   ┌─────────────────┐         ┌─────────────────┐   │
  │   │ TSC (core freq) │         │ APIC Timer      │   │
  │   │ 2-5 GHz         │         │ (bus freq)      │   │
  │   │                 │         │ 100-400 MHz     │   │
  │   │ TSC-Deadline ◄──┤         ├──► One-shot      │   │
  │   │ 用这个时钟       │         │    用这个时钟    │   │
  │   └─────────────────┘         └─────────────────┘   │
  │         ↑                            ↑              │
  │    分辨率更高                   分辨率更低            │
  │    频率恒定                    频率可能变化           │
  │    (Invariant TSC)            (受平台约束)           │
  └─────────────────────────────────────────────────────┘
```

- **One-shot** 的递减计数器由 **bus clock** 驱动，这是 CPU 外部互连总线的时钟，频率远低于核心频率
- **TSC-Deadline** 直接使用 **CPU core clock** 驱动的 TSC，频率通常是 bus clock 的 10-50 倍

### 4.2 编程接口差异

```
One-shot 编程是"相对定时"：
  ─────────────────────────────────────────→ 时间轴
       │                          │
     write(ICR, N)              触发
       │← N × (1/bus_freq) ────→│
       
  问题：从调用 lapic_next_event() 到 MMIO 写入完成之间
  有不确定延迟，这段时间不计入 N，导致实际定时偏长。

TSC-Deadline 编程是"绝对定时"：
  ─────────────────────────────────────────→ 时间轴 (TSC)
       │              │          │
     rdtsc()         wrmsr()    触发
       │              │          │
    tsc_now        设置 target  TSC=target
       │←  delta  ──────────→│

  优势：目标值是绝对 TSC 值，不受编程操作本身耗时影响。
  rdtsc 到 wrmsr 之间消耗的时间已包含在 (tsc + delta) 中，
  只要 delta 计算正确，触发时刻精确。
```

**关键区别：One-shot 的"相对计时"受编程延迟影响，TSC-Deadline 的"绝对计时"天然免疫。**

但需注意一个微妙点：`lapic_next_deadline()` 中 `rdtsc()` 和 `wrmsrl()` 之间存在时间窗口，如果这个窗口内 delta 已经很小，可能导致设置 deadline 时 TSC 已超过目标值。不过这只影响极短定时场景（< ~50 ns），实际内核使用中不会出现。

### 4.3 校准误差的影响

One-shot 模式需要知道 bus clock 频率才能将纳秒数转换为计数值。内核通过以下方式获取 bus clock 频率：

```
启动时校准流程：
  1. setup_boot_APIC_clock()
     → calibrate_APIC_clock()
       → 使用 PIT 或 PM-Timer 作为参考源
       → 编程 LAPIC timer 一个已知时间段
       → 读取 LAPIC 计数器消耗了多少 tick
       → 计算 bus_clock_frequency

  校准精度受限于：
    - PIT 本身的频率精度（1.193182 MHz ± 几十 ppm）
    - 中断延迟引入的测量误差
    - 仅在启动时校准一次，运行时不再修正

  典型校准误差：0.01% - 0.1%（100-1000 ppm）
```

TSC-Deadline 模式不需要 bus clock 校准——TSC 频率通过 `CPUID leaf 0x15` 或 `MSR_PLATFORM_INFO` 等架构化接口**精确获取**，且 Invariant TSC 保证频率恒定。

### 4.4 分频器的量化效应

One-shot 模式的 Divide Configuration Register (DCR) 只支持 8 种分频比：

```
DCR 值:  0000 → div 2     0001 → div 4
         0010 → div 8     0011 → div 16
         1000 → div 32    1001 → div 64
         1010 → div 128   1011 → div 1

Linux 内核通常使用 div=1（APIC_TDR_DIV_1）以获得最高精度。
但即使 div=1，分辨率仍受限于 bus_clock 频率。
```

TSC-Deadline 没有分频器概念——直接在 TSC 粒度上操作，这是硬件设计上的根本简化。

---

## 5. Linux 内核中的实现差异

### 5.1 clockevents 注册

内核为 LAPIC timer 注册的 `clock_event_device` 会根据模式提供不同的 `set_next_event` 回调：

```c
// arch/x86/kernel/apic/apic.c

// One-shot 模式的回调
static int lapic_next_event(unsigned long delta,
                            struct clock_event_device *evt)
{
    apic_write(APIC_TMICT, delta);  // 写入 Initial Count (MMIO)
    return 0;
}

// TSC-Deadline 模式的回调
static int lapic_next_deadline(unsigned long delta,
                               struct clock_event_device *evt)
{
    u64 tsc = rdtsc();
    wrmsrl(MSR_IA32_TSC_DEADLINE,
           tsc + (((u64) delta) * TSC_DIVISOR));
    return 0;
}
```

两个函数的核心区别：

| 方面 | `lapic_next_event` | `lapic_next_deadline` |
|------|--------------------|-----------------------|
| I/O 类型 | MMIO (apic_write) | MSR (wrmsrl) |
| 时间语义 | 相对值（从现在起 N ticks） | 绝对值（TSC 到达此值） |
| 时基 | bus clock / divider | TSC frequency |
| delta 含义 | bus clock ticks | TSC cycles (经 TSC_DIVISOR 换算) |

### 5.2 精度特征（rating）

内核通过 `clock_event_device.rating` 反映时钟源质量：

```c
// 简化的注册逻辑
if (cpu_has(X86_FEATURE_TSC_DEADLINE_TIMER)) {
    lapic_clockevent.features |= CLOCK_EVT_FEAT_TSC_DEADLINE;
    lapic_clockevent.set_next_event = lapic_next_deadline;
    lapic_clockevent.rating = 600;  // 高优先级
} else {
    lapic_clockevent.set_next_event = lapic_next_event;
    lapic_clockevent.rating = 100;  // 标准优先级
}
```

TSC-Deadline 模式的 rating (600) 远高于 One-shot 模式 (100)，反映了内核对其精度优势的认可。其他时钟源对比：HPET = 110, PIT = 4。

### 5.3 hrtimer 框架集成

在 `CONFIG_HIGH_RES_TIMERS=y` 配置下，内核的高精度定时器（hrtimer）框架直接利用 `clock_event_device`：

```
hrtimer_start(timer, ktime)
  → __hrtimer_start_range_ns()
    → hrtimer_reprogram()
      → tick_program_event()
        → clockevents_program_event()
          → lapic_next_deadline() 或 lapic_next_event()
```

**精度传导链：**

- hrtimer 框架内部以 `ktime_t`（纳秒）为单位管理超时
- 下发到 clockevent 设备时，需将纳秒转换为设备原生单位
- One-shot：ns → bus ticks（有取整误差）
- TSC-Deadline：ns → TSC cycles（取整误差极小，因为 TSC 频率高）

```
取整误差量化：

One-shot (bus_clock = 100 MHz, div = 1):
  1 tick = 10 ns
  设定 12345 ns → 1234 ticks = 12340 ns
  误差 = 5 ns（最大 ±10 ns）

TSC-Deadline (TSC = 2.5 GHz):
  1 cycle = 0.4 ns
  设定 12345 ns → 30862 cycles = 12344.8 ns
  误差 = 0.2 ns（最大 ±0.4 ns）

精度提升：~25 倍
```

---

## 6. 虚拟化场景下的精度影响

### 6.1 KVM 中的 Timer 模拟

在 KVM 虚拟化环境中，无论 Guest 使用 One-shot 还是 TSC-Deadline 模式，timer 操作都会触发 VM-Exit，由 KVM 在 host 侧模拟。此时两种模式的**裸精度差异被 VM-Exit 开销所掩盖**：

```
Guest 视角的 Timer 精度：

          ┌── Guest 内部精度（One-shot vs TSC-Deadline 差异）
          │     ~10 ns vs ~1 ns（差异 ~10 ns）
          │
精度损失 ──┤── VM-Exit/Entry 开销
          │     AMD SVM: ~700-1200 ns
          │     Intel VMX: ~500-800 ns
          │     （比模式本身的精度差异大 50-100 倍）
          │
          └── KVM hrtimer 模拟误差
                ~100-1000 ns
```

**结论：在标准 KVM 虚拟化中，One-shot 与 TSC-Deadline 的裸精度差异（~10 ns）相比 VM-Exit 开销（~1000 ns）可以忽略不计。**

但 TSC-Deadline 模式在虚拟化中仍有重要优势：
- 与 NO_HZ/tickless 配合更好，减少 timer 中断总量
- 支持 VMX Preemption Timer 加速（Intel 平台）
- 支持 Timer Advance 自适应机制
- 是 exitless timer 方案的基础

### 6.2 Timer Passthrough 场景

在 [exitless timer](../concepts/concept-exitless-timer.md) 的 Timer Passthrough 方案（如字节跳动方案）下，Guest 直接操作物理 LAPIC timer 硬件，VM-Exit 被完全消除。此时 **One-shot vs TSC-Deadline 的精度差异重新变得显著**：

```
Timer Passthrough 下的精度对比：

One-shot (直接硬件):
  精度 = bus_clock 级 ≈ 5-100 ns 抖动

TSC-Deadline (直接硬件):
  精度 = TSC 级 ≈ 0.4-2 ns 抖动

此时两者的精度差异就是裸硬件差异，
TSC-Deadline 优势明显。

这也是为什么 Timer Passthrough 方案仅支持 TSC-Deadline 模式——
既然消除了虚拟化开销，就应该使用最高精度的 timer 模式。
```

---

## 7. 实际影响场景分析

### 7.1 对内核定时精度的影响

| 场景 | One-shot 影响 | TSC-Deadline 影响 |
|------|-------------|------------------|
| `nanosleep()` / `clock_nanosleep()` | 精度受限于 bus tick（~10 ns 级量化） | 精度接近纳秒级 |
| 高精度定时器 (`hrtimer`) | 有效精度 ~100 ns | 有效精度 ~10 ns |
| 调度器 tick（CFS） | 几乎无影响（tick 周期 ms 级，误差可忽略） | 同左 |
| `perf_event` 采样 | 采样间隔有 ~10 ns 级抖动 | 采样间隔抖动 < 1 ns |
| 网络协议栈超时 | 无明显影响（超时通常 ms-s 级） | 同左 |

### 7.2 对延迟敏感工作负载的影响

```
金融交易 / HFT 场景：
  ├── 订单执行间隔: 微秒级
  ├── 定时器精度需求: ~100 ns
  ├── One-shot: 可满足但有 ~10-100 ns 抖动
  └── TSC-Deadline: 轻松满足，抖动 < 10 ns

实时控制系统 (PREEMPT_RT)：
  ├── 控制循环周期: 10-1000 μs
  ├── 定时器精度需求: ~1 μs
  ├── One-shot: 可满足
  └── TSC-Deadline: 可满足且裕量更大

通用服务器工作负载：
  ├── 定时器精度需求: ~1 ms
  ├── One-shot: 完全满足
  └── TSC-Deadline: 完全满足（能力远超需求）
```

### 7.3 CPU 功耗角度

两种模式在功耗特性上基本等价——两者都支持 `NO_HZ` tickless 模式，空闲时不产生多余中断。但 TSC-Deadline 有微小优势：

- TSC 比较器是 CPU 核心内部的低功耗逻辑，不需要额外的总线时钟域硬件活跃
- One-shot 的递减计数器需要 bus clock 域持续驱动（尽管功耗增量极小）

---

## 8. 为什么现代系统全部使用 TSC-Deadline

TSC-Deadline 模式自 Intel Sandy Bridge (2011) 和 AMD Bulldozer (2011) 起被引入，到如今的 AMD EPYC (Zen 2+) 和 Intel Xeon (Ice Lake+) 已是标配。内核优先选择 TSC-Deadline 的原因总结：

1. **更高精度**：分辨率提高 10-50 倍，抖动减少 3-10 倍
2. **更简单的编程模型**：绝对时间语义，无需关心 bus clock 频率和分频器
3. **无校准需求**：TSC 频率通过 CPUID 精确获取，无需启动时校准
4. **与 NO_HZ_FULL 天然兼容**：只需在有定时需求时写一次 MSR
5. **虚拟化优化基础**：VMX Preemption Timer、Timer Advance、Exitless Timer 均基于 TSC-Deadline
6. **硬件趋势**：现代 CPU 的 Invariant TSC 保证频率恒定，是最可靠的时基

**回退场景**：仅在以下情况内核才使用 One-shot 模式：
- CPU 不支持 TSC-Deadline（`CPUID.01H:ECX[24]` = 0）
- TSC 被标记为不可靠（`tsc=unstable` 或检测到 TSC 不同步）
- 老旧 CPU（Sandy Bridge / Zen 之前）

---

## 9. 结论

| 维度 | One-shot | TSC-Deadline | 差异倍数 |
|------|----------|-------------|---------|
| 时基分辨率 | 5-100 ns | 0.2-0.5 ns | **10-200x** |
| 每次编程量化误差 | ±5-100 ns | ±0.2-0.5 ns | **25-200x** |
| 触发抖动 | 20-100 ns | 5-30 ns | **2-5x** |
| 编程延迟 | ~20-50 ns | ~12-20 ns | **~2x** |
| 校准引入的系统性误差 | 100-1000 ppm | ~0 | **显著** |
| 长定时累积漂移 | 可能 | 极小 | **显著** |

**核心结论**：

1. TSC-Deadline 模式在所有精度指标上全面优于 One-shot 模式，分辨率提升 10-200 倍，抖动降低 2-5 倍
2. 精度差异的根本原因是 **时钟域不同**（core TSC vs bus clock）和 **编程语义不同**（绝对值 vs 相对值）
3. 在裸机环境下，对于纳秒级精度需求的工作负载（HFT、实时控制），TSC-Deadline 具有实质性优势
4. 在标准 KVM 虚拟化环境下，VM-Exit 开销（~1 μs）远大于两种模式的精度差异（~10 ns），实际精度差异被掩盖
5. 在 Timer Passthrough 直通场景下，精度差异重新显现，这也是 passthrough 方案仅支持 TSC-Deadline 的原因之一
6. 现代 x86 CPU（AMD Zen 2+ / Intel Sandy Bridge+）全部支持 TSC-Deadline，内核默认优先使用

---

## See also

- [timing-subsystem](../entities/timing-subsystem.md) — 内核时钟子系统基础（jiffies, HZ, timer wheel）
- [analysis-epyc-lapic-timer](analysis-epyc-lapic-timer.md) — AMD EPYC 平台 LAPIC Timer 全面分析
- [concept-exitless-timer](../concepts/concept-exitless-timer.md) — Exit-less Timer 三厂商方案对比
- [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md) — KVM 中断虚拟化与 LAPIC 模拟
- [analysis-vm-exit-reduction-and-timer-virtualization](analysis-vm-exit-reduction-and-timer-virtualization.md) — VM Exit 消除与 Timer 虚拟化演进
