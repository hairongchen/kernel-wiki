---
type: analysis
created: 2026-04-10
updated: 2026-04-10
sources: [understanding-the-linux-kernel, professional-linux-kernel-architecture, qemu-kvm-source-code-and-application]
tags: [interrupts, apic, ipi, interrupt-delivery, smp, halt, idle, posted-interrupts]
---

# 中断传递流程

本分析追踪一个中断从硬件设备（或软件源）发出到目标 CPU 开始执行处理程序的完整生命周期。重点分析两个关键场景——目标 CPU **正在运行** 与 **处于停机/空闲状态**——因为二者的硬件和软件路径存在显著差异。

## 1. 中断源与路由概述

中断在传递之前，必须从源头路由到特定的 CPU。路径取决于中断控制器拓扑：

```
                        +-----------+
  设备 IRQ 引脚 ------->| I/O APIC  |---+
                        +-----------+   |    +------------+
                                        +--->| Local APIC |---> CPU 核心
  MSI/MSI-X 写操作 -------------------------| (每CPU一个) |
                                        +--->|            |
  IPI (来自另一个CPU) ----------------+    +------------+
                                             |
  本地中断源 (定时器, LINT, 错误) -----------+
```

### I/O APIC 路由决策

当设备在连接到 I/O APIC 的 IRQ 线上触发中断时，控制器查询该引脚对应的**重定向表**条目。条目指定：

| 字段 | 在传递中的作用 |
|------|--------------|
| `destination` | 目标 APIC ID——物理或逻辑寻址 |
| `dest_mode` | **物理模式**：单个 APIC ID。**逻辑模式**：与 LDR/DFR 匹配的位掩码 |
| `delivery_mode` | Fixed、Lowest Priority、NMI、SMI、INIT、ExtINT |
| `trigger_mode` | 边沿触发或电平触发——影响确认协议 |
| `vector` | 目标 CPU 使用的 IDT 向量号（32-255） |
| `mask` | 若置位则中断被屏蔽 |

对于 **Lowest Priority** 传递模式，I/O APIC（或在较新硬件上由 CPU 仲裁逻辑）选择任务优先级寄存器（TPR）值最低的 CPU——即当前执行最低优先级可中断工作的处理器。

### MSI 旁路

MSI/MSI-X 中断直接在内存写事务中编码目标 APIC ID 和向量号。写操作目标地址为 `0xFEE0_0000`（LAPIC MMIO 区域），芯片组直接将其传递到指定的 Local APIC，完全绕过 I/O APIC。

## 2. Local APIC 中断接受

无论来源（I/O APIC、MSI、IPI 还是本地中断），中断都到达目标 CPU 的 **Local APIC**。接受逻辑经过以下阶段：

### 阶段 1：IRR——记录请求

Local APIC 拥有一个 256 位的**中断请求寄存器（IRR）**，每个向量对应一位。当中断消息到达时，APIC 设置 IRR 中对应的位。对于边沿触发中断，在上升沿设置；对于电平触发中断，只要源保持有效就持续设置。

### 阶段 2：优先级过滤（TPR/PPR）

APIC 将 IRR 中最高优先级待处理向量的优先级与**处理器优先级寄存器（PPR）**进行比较：

```
PPR = max(TPR, ISR 中最高优先级向量)
```

- **TPR**（任务优先级寄存器）：由软件控制。操作系统设置此值以屏蔽低于某一优先级类的中断。高 4 位定义优先级类（0-15）；同一类内的向量共享相同优先级（每类跨越 16 个向量）。
- **ISR**（服务中寄存器）：跟踪当前正在服务的向量。

如果待处理向量的优先级类**高于** PPR 的优先级类，则中断有资格被传递。否则，它保持在 IRR 中挂起，直到 PPR 降低（例如在 EOI 之后）。

### 阶段 3：向 CPU 核心传递

一旦向量通过优先级过滤器，APIC 向 CPU 核心断言 **INTR 引脚**（或其在内部总线上的逻辑等价物）。接下来发生的事情取决于 CPU 是正在运行还是已停机。

## 3. 场景 A：目标 CPU 正在运行

这是系统正常运行期间最常见的场景。CPU 正在执行指令（内核代码、用户空间代码或另一个中断处理程序）。

### 3A.1——INTR 信号与 EFLAGS.IF

CPU 在**指令边界**处检查 INTR 信号（在完成一条指令和取下一条指令之间）。必须满足两个条件：

1. **EFLAGS.IF = 1**（中断已启用）。如果 IF 被清除（例如在 `cli`/`sti` 临界区内或在另一个中断处理程序的上半部期间），CPU 忽略 INTR。中断保持在 LAPIC 的 IRR 中挂起，直到 IF 被重新设置。
2. 没有更高优先级的事件挂起（NMI、SMI 或调试异常具有更高优先级）。

```
指令 N 执行完成
    |
    v
[检查 INTR 且 IF==1?]
    |           |
   是           否 ---> 继续执行指令 N+1
    |                   (中断留在 IRR 中)
    v
开始中断接受
```

### 3A.2——硬件中断接受序列

当 CPU 接受中断时，以下硬件序列**原子地**执行（无需软件参与）：

1. **确认 LAPIC**：CPU 从 LAPIC 读取中断向量（中断确认周期）。LAPIC 执行：
   - 清除 **IRR** 中对应的位
   - 设置 **ISR**（服务中寄存器）中对应的位
   - 如果还有更高优先级的待处理中断，则保持 INTR 断言

2. **将被中断的上下文保存到栈上**：
   - 压入 `SS` 和 `ESP`（如果特权级改变，即用户态→内核态切换）
   - 压入 `EFLAGS`
   - 压入 `CS`
   - 压入 `EIP`（**下一条**指令的地址——即本应执行的那条）
   - 如果 IDT 条目是**中断门**：清除 `EFLAGS.IF`（禁用后续可屏蔽中断）
   - 如果 IDT 条目是**陷阱门**：保持 `EFLAGS.IF` 不变

3. **查找 IDT**：以向量号为索引查找 256 项 IDT 中的门描述符。

4. **加载处理程序**：将 `CS:EIP` 设置为门描述符中的处理程序地址。如果发生特权级切换，则从 TSS 切换到内核栈。

### 3A.3——软件处理路径（Linux）

硬件中断的 IDT 条目（向量 32-255）指向汇编入口桩，汇聚到通用路径：

```
入口桩 (向量特定)
  |
  v
common_interrupt:
  SAVE_ALL                    /* 压入所有通用寄存器 */
  |
  v
do_IRQ(struct pt_regs *regs):
  irq_enter()                 /* preempt_count += HARDIRQ_OFFSET */
  |
  v
  irq_desc[irq].handle_irq() /* genirq 流处理程序 */
    |
    +-- handle_level_irq()  或  handle_edge_irq()  或  handle_fasteoi_irq()
    |     |
    |     v
    |   chip->ack() / chip->mask_ack()
    |   遍历 irqaction 链：调用每个 handler(irq, dev_id)
    |   chip->unmask() / chip->eoi()
    |
  irq_exit()                  /* preempt_count -= HARDIRQ_OFFSET */
    |                         /* 若有软中断挂起：do_softirq() */
    v
ret_from_intr:
  若返回用户态：
    检查 TIF_NEED_RESCHED  --> schedule()
    检查 TIF_SIGPENDING    --> do_signal()
  若返回内核态：
    检查 preempt_count == 0 且 TIF_NEED_RESCHED --> preempt_schedule_irq()
  RESTORE_ALL
  iret
```

### 3A.4——延迟特性（运行中的 CPU）

| 阶段 | 大致延迟 | 说明 |
|------|---------|------|
| I/O APIC → LAPIC 消息 | ~1 us | 芯片间总线消息 |
| LAPIC 仲裁 + IRR 设置 | ~100 ns | 片上逻辑 |
| 等待指令边界 | 0 — ~100 ns | 取决于当前指令 |
| IF 检查 + 接受 + 上下文保存 | ~50-100 ns | 硬件微码 |
| IDT 查找 + 跳转到处理程序 | ~10-20 ns | 在 L1 缓存中 |
| SAVE_ALL + do_IRQ 入口 | ~100-200 ns | 软件开销 |
| **总计到处理程序入口** | **~1-2 us** | 典型值，随负载变化 |

### 3A.5——IF=0 的情况（中断已禁用）

如果 CPU 以 `IF=0` 运行——处于 `spin_lock_irqsave()` 临界区、另一个中断的上半部或显式 `local_irq_disable()` 区域内——中断保持在 LAPIC IRR 中挂起，不消耗 CPU 周期。

当中断被重新启用（`sti`、`local_irq_enable()`、`spin_unlock_irqrestore()`）时，CPU 立即在下一个指令边界检查 INTR 并开始接受序列。`sti` 指令有一个**单指令延迟**：CPU 直到 `sti` 之后的下一条指令完成后才会识别 INTR（即"STI 影子"）。这允许如下代码：

```asm
sti
hlt     ; 在中断启用的状态下原子地进入停机
```

这对于下面描述的停机 CPU 场景至关重要。

## 4. 场景 B：目标 CPU 已停机（空闲）

当 CPU 没有可运行的任务时，Linux 空闲循环将其置于低功耗停机状态。这是轻负载系统的常见状态——在运行单线程工作负载的 64 核机器上，通常有 63 个核心处于停机状态。

### 4B.1——空闲循环与 HLT

Linux 空闲循环（简化版）如下：

```c
void cpu_idle(void) {
    while (1) {
        while (!need_resched()) {
            /* 进入 C-state（停机或更深层） */
            local_irq_disable();
            if (!need_resched())
                safe_halt();    /* sti; hlt —— 原子操作 */
            else
                local_irq_enable();
        }
        schedule();   /* 选择下一个任务 */
    }
}
```

`safe_halt()` 宏展开为 `sti; hlt` 指令序列。x86 架构保证 **STI 影子**阻止在 `sti` 和 `hlt` 之间识别中断，因此 CPU 原子地过渡到：

- **IF = 1**（中断已启用）
- **停机状态**（不取指令，最低功耗）

这种原子性至关重要。若没有它，将存在竞态条件：在 `sti` 和 `hlt` 之间到达的中断会被处理（清除唤醒需求），然后 `hlt` 执行，可能导致 CPU 无限期停机直到下一个中断到来。

#### 更深的 C-State（ACPI）

现代处理器支持比 HLT (C1) 更深的 C-state：
- **C1 (HLT)**：时钟门控。唤醒延迟 ~1 us。
- **C1E**：增强型停机。唤醒延迟 ~10 us。
- **C3 (Sleep)**：L1/L2 缓存可能被刷新。唤醒延迟 ~100 us。
- **C6 (Deep Power Down)**：核心电压降低。唤醒延迟 ~200+ us。

`cpuidle` 调度器根据预期空闲时长选择合适的 C-state。更深的状态节省更多功耗，但具有更高的**退出延迟**，直接影响中断响应时间。

### 4B.2——唤醒序列

当中断到达停机的 CPU 时：

```
LAPIC 接收中断消息
    |
    v
IRR 位设置，优先级检查通过
    |
    v
LAPIC 向 CPU 核心断言 INTR
    |
    v
[CPU 已停机——特殊唤醒路径]
    |
    v
CPU 退出停机状态（C-state 退出）：
  - 恢复时钟 (C1: ~1 us)
  - 恢复缓存 (C3: ~100 us)
  - 恢复电压 (C6: ~200 us)
    |
    v
在 hlt 之后的指令处恢复执行
    |
    v
CPU 检查 INTR（IF 已经为 1，来自 hlt 之前的 sti）
    |
    v
正常中断接受序列（同 3A.2）
  - 确认 LAPIC，IRR→ISR
  - 保存上下文，查找 IDT
  - 跳转到处理程序
```

关键洞察：**HLT 并非中断传递的障碍**——它被专门设计为可被中断（和 NMI）打断。`HLT` 指令的定义是：当 INTR（或 NMI）被断言时，在下一条指令处恢复执行（对于可屏蔽中断需要 IF=1）。

### 4B.3——延迟特性（停机的 CPU）

| 阶段 | C1 (HLT) | C3 (Sleep) | C6 (Deep) |
|------|-----------|------------|-----------|
| I/O APIC → LAPIC | ~1 us | ~1 us | ~1 us |
| C-state 退出 | ~1 us | ~50-100 us | ~100-200+ us |
| HLT 后中断接受 | ~100 ns | ~100 ns | ~100 ns |
| 软件处理程序入口 | ~300 ns | ~300 ns | ~300 ns |
| **总计** | **~2-3 us** | **~50-100 us** | **~100-200+ us** |

C-state 退出延迟在深睡眠状态下**占主导地位**。这是延迟敏感型工作负载（网络、实时系统）配置 `idle=poll`（永不停机）或通过 `/dev/cpu_dma_latency` 或内核参数（`processor.max_cstate`）限制 C-state 深度的主要原因。

### 4B.4——唤醒后的空闲循环

中断处理程序通过 `iret` 返回后，执行在 `hlt` 之后的指令处恢复，回到空闲循环。循环重新检查 `need_resched()`：

- 如果中断处理程序标记了一个任务为可运行状态（例如网络数据包唤醒了一个阻塞的进程），`TIF_NEED_RESCHED` 被设置，`schedule()` 拿起新任务。
- 如果不需要重新调度，CPU 重新进入停机状态。

## 5. 处理器间中断（IPI）

IPI 是一个 CPU 中断另一个 CPU 的机制。其关键用途包括：

| 用途 | 向量/函数 | 目标状态是否重要？ |
|------|----------|-------------------|
| **TLB 刷新** | `INVALIDATE_TLB_VECTOR` | 是——停机的 CPU 没有有效 TLB 条目，可能跳过 |
| **重新调度** | `RESCHEDULE_VECTOR` / `smp_send_reschedule()` | 是——必须唤醒停机的 CPU |
| **函数调用** | `CALL_FUNCTION_VECTOR` / `smp_call_function()` | 是——必须唤醒以执行 |
| **停止** | `smp_send_stop()` | 是——必须唤醒以停止 |

### 发送 IPI

IPI 通过写入发送方 Local APIC 的**中断命令寄存器（ICR）**来发送：

```c
/* 简化：向特定 CPU 发送 IPI */
apic_write(APIC_ICR2, target_apic_id << 24);  /* 目标 */
apic_write(APIC_ICR, vector | APIC_DEST_PHYSICAL | APIC_DM_FIXED);
```

LAPIC 通过系统总线（或在旧系统上通过 APIC 总线）向目标 LAPIC 发送 IPI 消息。传递遵循与其他中断相同的 IRR → 优先级检查 → INTR 断言路径。

### IPI 发送到运行中的 CPU

目标 CPU 在下一个指令边界被中断（假设 IF=1）。IPI 处理程序运行，执行请求的操作（刷新 TLB、设置 `TIF_NEED_RESCHED`、执行函数），然后返回。被中断的代码透明地恢复执行。

### IPI 发送到停机的 CPU

IPI 将目标 CPU 从停机状态唤醒。C-state 退出后，IPI 处理程序执行。这是调度器唤醒空闲 CPU 来运行新就绪任务的方式：

```c
/* kernel/sched/core.c —— 简化 */
void wake_up_process(struct task_struct *p) {
    /* ... 选择目标 CPU ... */
    if (task_cpu(p) != smp_processor_id()) {
        /* 目标在另一个 CPU 上——可能已停机 */
        smp_send_reschedule(task_cpu(p));  /* IPI */
    }
}
```

重新调度 IPI 处理程序仅设置 `TIF_NEED_RESCHED`。当处理程序返回到空闲循环时，`need_resched()` 返回 true，`schedule()` 拿起新唤醒的任务。

## 6. 边沿触发 vs. 电平触发传递

触发模式影响中断源和 LAPIC 之间的传递协议：

### 边沿触发

```
信号: ___/‾‾‾‾\___

在上升沿（0→1 转换）设置 IRR
一旦设置在 IRR 中，源可以撤销断言
在 IRR 位已设置时的第二个边沿：丢失（APIC 中无排队）
```

- 使用者：MSI/MSI-X、传统 ISA 设备（通常）
- 风险：如果设备在处理程序清除源之前重新断言，会**丢失中断**。驱动程序必须检查设备状态寄存器，并在处理后重新检查。
- genirq 流：`handle_edge_irq()`——先确认，再处理。如果处理期间有新的边沿到达，处理程序重新运行。

### 电平触发

```
信号: ___/‾‾‾‾‾‾‾‾‾‾‾\___
          ^            ^
          IRR 设置     EOI 后 + 设备撤销
                       后 IRR 清除
```

- 使用者：PCI 传统设备（共享线路）
- 只要线路被断言，LAPIC 的 IRR 位保持设置
- EOI 后，如果线路仍被断言（设备尚未被服务），IRR 位立即重新设置，触发另一个中断
- genirq 流：`handle_level_irq()`——屏蔽+确认，再处理，然后取消屏蔽。处理程序执行期间的屏蔽可防止中断风暴。

### 对停机 CPU 的影响

两种触发模式唤醒停机 CPU 的方式相同——LAPIC 无论如何都断言 INTR。差异仅在 CPU 唤醒并处理中断后的**确认协议**中体现。

## 7. 完整流程——并排对比

```
                    运行中的 CPU                        停机的 CPU
                    (假设 IF=1)                         (在 HLT 中, IF=1)
                         |                                    |
  中断到达               |                                    |
  Local APIC             |                                    |
         |               |                                    |
         v               v                                    v
    设置 IRR 位     设置 IRR 位                          设置 IRR 位
         |               |                                    |
         v               v                                    v
    优先级>PPR?     在指令边界                            断言 INTR
    是：断言        检查 INTR                            (CPU 已停机)
    INTR                 |                                    |
         |               v                                    v
         |          IF==1?                               C-state 退出
         |          是：接受                              (1us — 200+us)
         |               |                                    |
         |               v                                    v
         |          IRR→ISR                              在 HLT 后恢复
         |          压入上下文                                 |
         |          查找 IDT                                  v
         |          跳转到处理程序                        检查 INTR (IF=1)
         |               |                               接受中断
         |               v                                    |
         |          do_IRQ()                                   v
         |          irqaction 链                          同运行状态：
         |          irq_exit()                            IRR→ISR, 压入上下文,
         |          ret_from_intr                         IDT, 处理程序
         |               |                                    |
         |               v                                    v
         |          恢复被中断                            do_IRQ() 等
         |          的代码                                     |
         |                                                    v
         |                                              ret_from_intr
         |                                              → 空闲循环
         |                                              → need_resched()?
         |                                              → schedule() 或
         |                                                重新进入 HLT
```

## 8. 实际应用

### 延迟调优

| 调优旋钮 | 效果 |
|---------|------|
| `idle=poll` | 永不停机——最低延迟，最高功耗 |
| `processor.max_cstate=1` | 仅使用 C1 (HLT)——~1us 退出延迟 |
| `/dev/cpu_dma_latency` | PM QoS 约束——cpuidle 调度器遵循 |
| `irqbalance` 守护进程 | 在 CPU 间分配 IRQ——避免唤醒错误的 CPU |
| `/proc/irq/N/smp_affinity` | 将 IRQ 固定到特定 CPU——避免停机的目标 |
| `isolcpus` + `irqaffinity` | 将 CPU 专用于特定工作负载 |

### 唤醒-然后-调度模式

内核中的常见模式：设备中断到达 → 处理程序处理数据 → 唤醒阻塞进程 → 如果该进程所在的 CPU 已停机，发送重新调度 IPI。这涉及**两次**中断传递：

1. 设备中断 → CPU A（IRQ 路由到的位置）
2. 重新调度 IPI → CPU B（目标任务上次运行的位置，可能已停机）

智能 IRQ 亲和性配置可以将此减少为一次传递——将设备中断路由到等待进程所睡眠的同一 CPU。

### KVM/虚拟化考量

在虚拟化环境中，向客户机 VCPU 传递中断增加了额外层次。参见 [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md) 了解 posted-interrupt 机制，它处理两种场景：

- **VCPU 在客户模式中运行**：Posted-interrupt 通知 IPI 使硬件将 `pir[]` 位合并到虚拟 VIRR 中——无需 VM Exit。
- **VCPU 未运行（已被调度出）**：待处理中断在下次 VM Entry 时通过 `vmx_sync_pir_to_irr()` 从 `pir[]` 同步到 IRR。

## 参见

- [interrupt-handling](../entities/interrupt-handling.md) — IDT、PIC/APIC 架构、软中断、tasklet、工作队列
- [concept-napi-interrupt-mitigation](../concepts/concept-napi-interrupt-mitigation.md) — 网络的中断到轮询转换
- [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md) — APICv/posted interrupts 的虚拟中断传递
- [concept-kernel-synchronization](../concepts/concept-kernel-synchronization.md) — 受中断上下文影响的同步原语
- [process-scheduler](../entities/process-scheduler.md) — 调度器与中断驱动唤醒的交互
- [timing-subsystem](../entities/timing-subsystem.md) — 定时器中断作为本地 APIC 传递的特殊案例
- [analysis-interrupt-delivery-process](analysis-interrupt-delivery-process.md) — English version
