---
type: analysis
created: 2026-04-11
updated: 2026-04-11
sources: [understanding-the-linux-kernel, professional-linux-kernel-architecture, qemu-kvm-source-code-and-application, lcna-co2012-sekiyama, minimizing-vmexits-pv-ipi-passthrough-timer]
tags: [interrupts, exceptions, pic, apic, msi, ipi, softirq, tasklets, work-queues, genirq, kvm, interrupt-virtualization]
---

# Linux 内核中断类型总览

本文从多个维度对 Linux 内核中的中断进行系统性分类，涵盖硬件中断、异常、触发模式、门类型、上下半部机制以及 KVM 虚拟化场景下的中断路径。

## 1. 按同步性分类

| 类型 | 触发方式 | 示例 |
|------|---------|------|
| **异步中断（硬件中断）** | 外部硬件设备随机发出 | 网卡、磁盘、键盘 |
| **同步中断（异常）** | CPU 执行指令时内部产生 | 缺页异常、除零、断点 |

异步中断在任意指令边界被 CPU 检测到（需要 `EFLAGS.IF=1`），而同步中断（异常）在引发问题的指令执行过程中立即产生。

## 2. 异常的三个子类

异常按照 CPU 保存的 `EIP` 位置和可恢复性分为三类：

| 子类 | 保存的 EIP | 行为 | 示例 |
|------|-----------|------|------|
| **Fault（故障）** | 指向**引发故障的指令** | 可修复后**重新执行**该指令 | Page Fault (#14)、GPF (#13)、Invalid Opcode (#6) |
| **Trap（陷阱）** | 指向**下一条指令** | 修复后继续执行 | Breakpoint (#3)、Overflow (#4)、Debug (#1) |
| **Abort（中止）** | 不确定 | 不可恢复，进程或系统终止 | Machine Check (#18)、Double Fault (#8) |

Page Fault (#14) 是最关键的 Fault 类异常——它是 [copy-on-write](../concepts/concept-copy-on-write.md) 和需求分页（demand paging）的核心机制。大多数其他异常处理程序会向当前进程发送信号（如 `SIGSEGV`、`SIGFPE`）。

### 关键异常向量表

| 向量 | 名称 | 类型 | 处理函数 |
|------|------|------|---------|
| 0 | Divide Error | Fault | `divide_error()` |
| 1 | Debug | Trap/Fault | `debug()` |
| 3 | Breakpoint | Trap | `int3()` — 系统门 (DPL=3) |
| 6 | Invalid Opcode | Fault | `invalid_op()` |
| 7 | Device Not Available | Fault | `device_not_available()` — 触发惰性 FPU 恢复 |
| 13 | General Protection | Fault | `general_protection()` |
| 14 | Page Fault | Fault | `page_fault()` → `do_page_fault()` |
| 18 | Machine Check | Abort | `machine_check()` |

向量 0-31 由 Intel 保留用于异常，向量 32-255 可用于硬件中断和软件定义用途。

## 3. 按中断控制器分类（硬件中断）

### 3.1 PIC（8259A 可编程中断控制器）

两片级联的 8259A 芯片，提供 15 条可用 IRQ 线（IRQ 0-15，IRQ 2 用于级联）。每片维护三个核心寄存器：

- **IRR**（中断请求寄存器）——中断被断言时设置
- **ISR**（服务中寄存器）——中断正在被 CPU 处理时设置
- **IMR**（中断屏蔽寄存器）——用于屏蔽特定 IRQ 线

PIC 只能将中断路由到单个 CPU，仅适用于单处理器系统。

### 3.2 I/O APIC

系统级中断路由器，替代 PIC 用于多处理器系统。拥有 24 条重定向表项，每条指定：

| 字段 | 作用 |
|------|------|
| `vector` | IDT 向量号 (0-255) |
| `delivery_mode` | Fixed、Lowest Priority、NMI、SMI、INIT、ExtINT |
| `dest_mode` | 物理寻址（单个 APIC ID）或逻辑寻址（位掩码） |
| `trig_mode` | 边沿触发或电平触发 |
| `mask` | 屏蔽位 |
| `dest_id` | 目标 APIC ID |

### 3.3 Local APIC 本地中断

每个 CPU 一个 Local APIC，处理四类中断：

1. **本地中断**——LINT0、LINT1、性能计数器、热传感器、APIC 错误
2. **定时器中断**——支持 one-shot、periodic、TSC-deadline 三种模式
3. **处理器间中断（IPI）**——CPU 间通信：TLB 刷新、重新调度、函数调用广播
4. **外部中断**——来自 I/O APIC 或 MSI 的路由

LAPIC 寄存器空间映射在 `0xFEE00000`，占 4 KB。

### 3.4 MSI/MSI-X（消息信号中断）

MSI 完全绕过 I/O APIC。设备通过内存写事务（目标地址 `0xFEE0_0000`）直接向目标 LAPIC 发送中断，地址和数据字段中编码目标 APIC ID、传递模式和向量号。

| 优势 | 说明 |
|------|------|
| 无引脚共享 | 消除共享 IRQ 线路冲突 |
| 更低延迟 | 跳过 I/O APIC 路由步骤 |
| 更多向量 | MSI 每设备最多 32 个，MSI-X 最多 2048 个 |

### 3.5 IPI（处理器间中断）

通过写发送方 LAPIC 的 ICR（中断命令寄存器）发送：

| 用途 | 向量/函数 | 说明 |
|------|----------|------|
| TLB 刷新 | `INVALIDATE_TLB_VECTOR` | 通知其他 CPU 使 TLB 条目无效 |
| 重新调度 | `RESCHEDULE_VECTOR` | 唤醒空闲 CPU 运行新任务 |
| 函数调用 | `CALL_FUNCTION_VECTOR` | `smp_call_function()` 跨 CPU 执行 |
| 系统停止 | `smp_send_stop()` | 停止所有 CPU |

## 4. 按触发模式分类

### 边沿触发（Edge-triggered）

```
信号: ___/‾‾‾‾\___
         ^
         IRR 在 0→1 上升沿时设置
```

- **使用者**：MSI/MSI-X、传统 ISA 设备
- **风险**：IRR 位已设置时的第二个边沿会丢失——驱动程序必须检查设备状态并重新检查
- **genirq 流处理程序**：`handle_edge_irq()` — ack → handle → 重新检查是否有新边沿

### 电平触发（Level-triggered）

```
信号: ___/‾‾‾‾‾‾‾‾‾‾‾\___
         ^                ^
         IRR 设置         EOI 后 + 设备撤销后 IRR 清除
```

- **使用者**：PCI 传统设备（共享线路）
- **特性**：线路保持有效期间 IRR 位持续设置；EOI 后若线路仍有效则立即重新触发
- **genirq 流处理程序**：`handle_level_irq()` — mask+ack → handle → unmask

其他 genirq 流处理程序：
- `handle_fasteoi_irq()` — handle → eoi（现代 APIC，自动屏蔽）
- `handle_simple_irq()` — 仅 handle（软件生成的 IRQ）

## 5. 按 IDT 门类型分类

IDT（中断描述符表）有 256 个条目，每个条目是一个门描述符：

| 门类型 | IF 是否清除 | DPL | 用途 |
|--------|-----------|-----|------|
| **中断门（Interrupt Gate）** | 是（禁用后续可屏蔽中断） | 0 | 硬件 IRQ——防止同一 IRQ 线嵌套处理 |
| **陷阱门（Trap Gate）** | 否（中断保持开启） | 0 | 大多数异常——允许异常处理期间响应其他中断 |
| **系统门（System Gate）** | 否 | 3 | 用户态可触发：`int $0x80`（系统调用）、`int3`（断点） |

中断门与陷阱门的核心区别：进入中断门时硬件自动清除 `EFLAGS.IF`，确保上半部处理期间不被其他可屏蔽中断打断。

## 6. 按处理时机分类（上半部 / 下半部）

Linux 将中断处理分为**上半部**（立即执行，时间关键）和**下半部**（延迟执行，完成剩余工作）。

### 上半部（Top Half）

通过 `request_irq()` 注册的中断处理程序。在中断上下文中运行：
- 不能睡眠
- 不能访问用户空间
- 当前 IRQ 线已禁用
- 应做最少工作：确认硬件、拷贝数据、调度下半部

### 下半部机制对比

| 属性 | Softirq（软中断） | Tasklet | Work Queue（工作队列） |
|------|-----------------|---------|----------------------|
| **上下文** | 软中断上下文（原子） | 软中断上下文（原子） | 进程上下文（**可睡眠**） |
| **并发性** | 同一 softirq 可多 CPU 并行 | 同一 tasklet 单 CPU 串行 | 每 CPU 工作线程 |
| **分配** | 静态（编译时） | 动态（运行时） | 动态（运行时） |
| **自身加锁** | 必须自行处理 | 自动串行化 | 进程上下文规则 |
| **适用场景** | 网络、块 I/O 等高吞吐子系统 | 大多数驱动下半部 | 需要睡眠的延迟工作 |

#### 六种静态 Softirq 类型（Linux 2.6.11）

| 索引 | 名称 | 用途 |
|------|------|------|
| 0 | `HI_SOFTIRQ` | 高优先级 tasklet |
| 1 | `TIMER_SOFTIRQ` | 定时器回调 |
| 2 | `NET_TX_SOFTIRQ` | 网络发送 |
| 3 | `NET_RX_SOFTIRQ` | 网络接收 |
| 4 | `SCSI_SOFTIRQ` | SCSI 后处理 |
| 5 | `TASKLET_SOFTIRQ` | 普通优先级 tasklet |

Softirq 在三个时机被处理：硬件中断返回时（`irq_exit()`）、`ksoftirqd` 内核线程中、显式调用 `do_softirq()` 的代码中。高负载下 `ksoftirqd` 以 nice 19 运行，防止软中断饿死用户空间。

## 7. KVM 虚拟化中的中断路径

在虚拟化场景中，KVM 模拟或加速多种中断传递路径：

| 路径 | VM Exit 次数 | 机制 |
|------|-------------|------|
| **PIC 模拟** | 多次 | `kvm_pic_set_irq()` → IRR/ISR → VMCS 注入 |
| **I/O APIC 模拟** | 多次 | `ioapic_set_irq()` → `kvm_irq_delivery_to_apic()` → LAPIC IRR → VMCS 注入 |
| **MSI** | 1 次 | `kvm_set_msi()` → LAPIC IRR → VMCS 注入 |
| **APICv / Posted Interrupts** | **0 次** | `vmx_deliver_posted_interrupt()` → `pi_desc.pir[]` → 硬件合并到 VIRR |
| **Direct Delivery（CPU 隔离）** | **0 次** | 清除 VMCS External-interrupt-exiting → 设备 MSI 直达客户机 IDT |
| **NoExit PVIPI** | **0 次** | 暴露 `pi_desc` 给客户机 → 客户机 `wrmsr ICR` → 硬件 posted-interrupt |
| **IRQfd** | 内核内完成 | eventfd → `kvm_set_irq()` → GSI 路由 → 控制器 → 客户机 |

### APICv 关键能力

| VMCS 控制位 | 效果 |
|------------|------|
| Virtualize APIC accesses | 客户机 APIC MMIO 访问重定向到虚拟 APIC 页 |
| APIC-register virtualization | 大多数 APIC 寄存器读取无需 VM Exit |
| TPR shadow | TPR 读写无需 VM Exit |
| Virtual-interrupt delivery | 处理器自动评估和传递虚拟中断 |
| Process posted interrupts | 处理器自动处理 posted-interrupt 通知 |

## 8. 完整分类维度图

```
Linux 内核中断
├── 按同步性
│   ├── 异步中断（硬件中断）
│   └── 同步中断（异常）
│       ├── Fault（可重新执行）
│       ├── Trap（继续执行）
│       └── Abort（不可恢复）
│
├── 按控制器
│   ├── PIC (8259A) — 15 IRQ, 单 CPU
│   ├── I/O APIC — 24 重定向项, 多 CPU
│   ├── Local APIC — 本地中断 + 定时器 + IPI
│   ├── MSI/MSI-X — 绕过 I/O APIC, 低延迟
│   └── IPI — CPU 间通信
│
├── 按触发模式
│   ├── 边沿触发 — 上升沿设置 IRR
│   └── 电平触发 — 持续有效期间设置 IRR
│
├── 按 IDT 门类型
│   ├── 中断门 — 清除 IF, DPL=0
│   ├── 陷阱门 — 保持 IF, DPL=0
│   └── 系统门 — 保持 IF, DPL=3
│
├── 按处理时机
│   ├── 上半部（Top Half）— 中断上下文, 不可睡眠
│   └── 下半部（Bottom Half）
│       ├── Softirq — 静态, 可多 CPU 并行
│       ├── Tasklet — 动态, 同 tasklet 串行
│       └── Work Queue — 进程上下文, 可睡眠
│
└── 虚拟化中断路径（KVM）
    ├── 软件模拟 — PIC / I/O APIC / LAPIC
    ├── APICv + Posted Interrupts — 零 VM Exit
    ├── Direct Delivery — CPU 隔离
    └── NoExit PVIPI — pi_desc 暴露
```

## 参见

- [interrupt-handling](../entities/interrupt-handling.md) — IDT、PIC/APIC 架构、softirq/tasklet/workqueue 详细实现
- [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md) — KVM 中断虚拟化全链路
- [analysis-interrupt-delivery-process-zh](analysis-interrupt-delivery-process-zh.md) — 中断传递流程深度分析（运行 vs 停机 CPU）
- [concept-napi-interrupt-mitigation](../concepts/concept-napi-interrupt-mitigation.md) — 网络中断缓解：中断+轮询混合模式
- [concept-kernel-synchronization](../concepts/concept-kernel-synchronization.md) — 中断上下文影响的同步原语
- [timing-subsystem](../entities/timing-subsystem.md) — 定时器中断作为 LAPIC 本地中断的特殊案例
- [analysis-vm-exit-reduction-and-timer-virtualization](analysis-vm-exit-reduction-and-timer-virtualization.md) — VM Exit 消除与 Timer 虚拟化优化
