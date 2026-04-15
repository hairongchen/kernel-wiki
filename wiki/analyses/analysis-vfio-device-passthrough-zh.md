---
type: analysis
created: 2026-04-10
updated: 2026-04-10
sources: [qemu-kvm-source-code-and-application, mastering-kvm-virtualization]
tags: [kvm, vfio, iommu, pci-passthrough, vt-d, amd-vi, sr-iov, device-assignment, dma-remapping]
---

# VFIO 与 IOMMU 设备直通

VFIO（Virtual Function I/O）是 Linux 内核框架，用于将物理设备安全地直接分配给用户空间程序——最重要的是分配给 QEMU/KVM 虚拟机。它依赖 IOMMU（I/O 内存管理单元）提供 DMA 隔离和中断重映射，确保直通设备只能访问为其显式映射的内存。

设备直通将虚拟机监控器从 I/O 数据路径中消除。与 QEMU 模拟虚拟设备或使用半虚拟化 virtio 不同，客户机 VM 使用其原生驱动程序直接驱动真实硬件。这提供了接近原生的 I/O 性能——对 GPU、高速网卡、NVMe 存储和 FPGA 加速器至关重要。

## IOMMU 硬件

### Intel VT-d 与 AMD-Vi

Intel 和 AMD 都提供 IOMMU 实现：

| 功能 | Intel VT-d | AMD-Vi (AMD IOMMU) |
|------|-----------|-------------------|
| DMA 重映射 | 多级页表 | 多级页表 |
| 中断重映射 | 中断重映射表 (IRT) | 中断重映射表 |
| I/O 缺页 | Posted Page Request (PPR) — VT-d 3.0 | PPR (AMD IOMMUv2) |
| 规范 | VT-d 规范 (Intel) | AMD I/O 虚拟化规范 |

两者提供相同的基本能力；Linux 通过统一的 IOMMU API（`drivers/iommu/`）对其进行抽象。

### DMA 重映射

没有 IOMMU 的情况下，执行 DMA 的 PCIe 设备可以读写系统上的任意物理地址。这对隔离性来说是灾难性的：分配给一个 VM 的设备可以 DMA 到另一个 VM 的内存或虚拟机监控器中。

IOMMU 在设备和物理内存之间插入一个**二级页表**，类似于 EPT/NPT 为 CPU 将客户机物理地址转换为宿主机物理地址的方式：

```
设备 DMA 请求
  │
  ▼
┌─────────────────┐
│  IOMMU           │  将设备总线地址 (IOVA)
│  页表            │  转换为宿主机物理地址 (HPA)
│                  │  使用每域页表
└─────────────────┘
  │
  ▼
宿主机物理内存
```

每个 IOMMU **域**有自己的一套页表。分配给 KVM 客户机的设备获得一个域，其页表映射客户机的 RAM——设备看到的 IOVA（I/O 虚拟地址）对应于客户机物理地址，IOMMU 将其转换为宿主机物理地址。设备无法访问此映射之外的任何内存。

### 中断重映射

没有中断重映射的情况下，恶意或有缺陷的设备可以伪造 MSI/MSI-X 中断消息，向任意 CPU 或向量注入中断——实质上是通过中断路径的 DMA 攻击。中断重映射强制所有设备产生的中断通过由虚拟机监控器维护的**中断重映射表**（IRT）：

1. 设备向中断地址范围生成 MSI/MSI-X 写操作。
2. IOMMU 拦截该写操作，使用消息的句柄字段索引 IRT。
3. IRT 条目指定实际的目标 CPU、向量和传递模式。
4. IOMMU 根据 IRT 条目重新格式化并传递中断。

这防止设备将中断定向到未授权的 CPU 或向量。

### 启动时配置

IOMMU 支持必须通过内核引导参数启用：

```
# Intel 系统
intel_iommu=on

# AMD 系统
amd_iommu=on

# 直通模式：仅对显式分配给 VM 的设备使用 IOMMU，
# 不对宿主机设备 DMA 使用（避免宿主机驱动的性能开销）
iommu=pt
```

推荐 KVM 宿主机使用 `iommu=pt` 选项。它为宿主机设备配置 IOMMU 为直通模式（无转换开销），同时仍为通过 VFIO 分配的设备启用完整的 IOMMU 隔离。

验证 IOMMU 是否激活：

```bash
dmesg | grep -i iommu
# Intel: "DMAR: IOMMU enabled"
# AMD:   "AMD-Vi: AMD IOMMUv2 functionality not available on this system" 或类似信息
```

## IOMMU 组

### 隔离边界

IOMMU 组是 IOMMU 能够与所有其他设备隔离的最小设备集合。它是设备分配的基本单位——你不能从一个组中分配单个设备；整个组必须一起分配。

IOMMU 组之所以存在，是因为 PCIe 拓扑允许设备之间进行绕过 IOMMU 的点对点事务：

```
                ┌─────────┐
                │  Root    │
                │ Complex  │
                └────┬────┘
                     │
              ┌──────┴──────┐
              │   PCIe      │
              │   Switch    │     ← 若交换机缺少 ACS，其后
              ├──────┬──────┤       的设备可以点对点 DMA，
              │      │      │       必须在同一组中
           ┌──┴──┐┌──┴──┐┌──┴──┐
           │设备A││设备B││设备C│
           └─────┘└─────┘└─────┘
              IOMMU 组 N
```

### 访问控制服务（ACS）

PCIe ACS 是一种能力，防止同一交换机或桥接器后面的设备之间进行点对点事务而不经过根联合体（IOMMU 所在位置）。如果 PCIe 交换机支持 ACS，内核可以将其后的每个设备放入各自独立的 IOMMU 组。没有 ACS 的话，交换机后的所有设备被分组在一起。

### 发现 IOMMU 组

```bash
# 列出所有 IOMMU 组
ls /sys/kernel/iommu_groups/

# 显示特定组中的设备
ls /sys/kernel/iommu_groups/14/devices/
# 0000:03:00.0  0000:03:00.1

# 查找设备属于哪个组
readlink /sys/bus/pci/devices/0000:03:00.0/iommu_group
# ../../../kernel/iommu_groups/14
```

多功能设备（例如具有不同 PCI 功能的网卡端口）通常将所有功能放在同一个组中。分配一个功能需要分配所有功能。

## VFIO 内核框架

VFIO 通过三层抽象提供安全的用户空间可访问设备接口：

```
┌─────────────────────────────────────┐
│  用户空间 (QEMU)                     │
│                                     │
│  Container fd (/dev/vfio/vfio)      │  ← IOMMU 上下文：DMA 映射
│    └── Group fd (/dev/vfio/<N>)     │  ← IOMMU 组：组内所有设备
│          └── Device fd              │  ← 单个设备：BAR、中断
└─────────────────────────────────────┘
```

### 容器

VFIO 容器（`/dev/vfio/vfio`）代表一个 IOMMU 上下文——一组 DMA 页表映射。打开容器设备并设置 IOMMU 类型（通常为 `VFIO_TYPE1_IOMMU`）建立转换上下文。容器是编程 DMA 映射的地方：

- `VFIO_IOMMU_MAP_DMA` — 将 IOVA 范围映射到用户空间虚拟地址（内核锁定页面并编程 IOMMU 页表）
- `VFIO_IOMMU_UNMAP_DMA` — 移除映射

### 组

每个 IOMMU 组由设备节点 `/dev/vfio/<group_id>` 表示。打开组 fd 并将其附加到容器，将该组中的所有设备链接到容器的 IOMMU 上下文。组中的所有设备必须：(a) 绑定到 VFIO 驱动（如 `vfio-pci`），或 (b) 绑定到已知安全的桩驱动，或 (c) 未绑定——之后该组才可使用。

### 设备

单个设备通过 `VFIO_GROUP_GET_DEVICE_FD` 从组中获取。设备 fd 暴露：

| ioctl | 用途 |
|-------|------|
| `VFIO_DEVICE_GET_INFO` | 区域（BAR）数量和中断类型 |
| `VFIO_DEVICE_GET_REGION_INFO` | 每个 BAR 的大小、偏移和标志 |
| `VFIO_DEVICE_GET_IRQ_INFO` | 中断类型（INTx、MSI、MSI-X）和数量 |
| `VFIO_DEVICE_SET_IRQS` | 配置中断传递（基于 eventfd） |
| `VFIO_DEVICE_RESET` | 重置设备 |

设备 BAR 可以通过设备 fd 的 `read()`/`write()` 访问（慢速，陷入式），或通过在 `VFIO_DEVICE_GET_REGION_INFO` 返回的偏移处进行 `mmap()` 访问（快速，用户空间直接 MMIO）。

### 将设备绑定到 vfio-pci

在 VFIO 管理设备之前，必须先将设备从其宿主机驱动解绑，然后绑定到 `vfio-pci` 驱动：

```bash
# 加载 VFIO 模块
modprobe vfio-pci

# 识别设备
lspci -nn -s 0000:03:00.0
# 03:00.0 Ethernet controller [0200]: Intel Corporation 82599ES [8086:10fb]

# 从宿主机驱动解绑
echo 0000:03:00.0 > /sys/bus/pci/devices/0000:03:00.0/driver/unbind

# 绑定到 vfio-pci（使用 vendor:device ID）
echo "8086 10fb" > /sys/bus/pci/drivers/vfio-pci/new_id

# 验证
ls -la /dev/vfio/
# 14  vfio    (组 14 已出现)
```

绑定后，设备不再对宿主机可用，可以分配给 VM。

## QEMU/KVM 集成

当 QEMU 以 `-device vfio-pci,host=0000:03:00.0` 启动时，发生以下序列：

### 1. 设备获取

```
QEMU 打开 /dev/vfio/vfio                    → 容器 fd
QEMU 在容器上设置 VFIO_TYPE1_IOMMU
QEMU 打开 /dev/vfio/<group_id>              → 组 fd
QEMU 将组附加到容器                           (VFIO_GROUP_SET_CONTAINER)
QEMU 从组获取设备 fd                          (VFIO_GROUP_GET_DEVICE_FD)
```

### 2. DMA 映射（客户机 RAM）

QEMU 通过 `VFIO_IOMMU_MAP_DMA` 将客户机的整个 RAM 映射到 IOMMU。每个 KVM 内存槽都被映射，使设备的 DMA 地址对应于客户机物理地址：

```
VFIO_IOMMU_MAP_DMA:
  iova  = guest_physical_address    （设备将 DMA 到的地址）
  vaddr = qemu_virtual_address      （QEMU 对客户机 RAM 的 mmap）
  size  = memory_region_size
```

内核锁定客户机 RAM 页面并编程 IOMMU 页表。设备现在可以使用客户机物理地址直接 DMA 到客户机内存——完全符合客户机原生驱动的预期。

### 3. BAR 映射

QEMU 通过 `VFIO_DEVICE_GET_REGION_INFO` 读取设备的 BAR 信息，并将 BAR 作为 MMIO 区域映射到客户机的物理地址空间。当客户机驱动访问设备寄存器时：

- **通过 mmap 映射的 MMIO BAR**：客户机访问通过 EPT 到达 QEMU 从 VFIO 设备 fd mmap 的 MMIO 区域。访问直接到达物理设备——数据平面寄存器访问不需要 VM Exit。
- **陷入式区域**：某些 BAR 区域（如 PCI 配置空间）被 KVM 陷入，导致 VM Exit，以便 QEMU 可以模拟或调解访问。

### 4. 中断传递

VFIO 使用与 KVM 的 irqfd 机制集成的 eventfd 配置设备中断：

```
物理设备触发 MSI-X 中断
  → VFIO 接收中断（通过宿主机的中断处理程序）
  → VFIO 发信号到已注册的 eventfd
  → KVM 的 irqfd 看到 eventfd 信号
  → KVM 向客户机注入中断
  → （使用 APICv/posted interrupts 时：无需 VM Exit 即可传递）
```

这与 [vhost](../entities/vhost.md) 使用的 irqfd 机制相同，提供低延迟中断传递，完全绕过 QEMU 的事件循环。

## SR-IOV 直通

SR-IOV（单根 I/O 虚拟化）扩展了直通模型，允许**多个 VM 共享单个物理设备**，同时保持硬件级隔离。

### 物理功能与虚拟功能

支持 SR-IOV 的设备暴露：

- **物理功能（PF）**：功能完整的 PCIe 功能。宿主机驱动管理 PF 来配置设备和创建 VF。
- **虚拟功能（VF）**：轻量级 PCIe 功能，每个都有自己的 BAR、MSI-X 向量和队列对。每个 VF 作为独立的 PCI 设备出现，可以通过 VFIO 分配给 VM。

```
┌────────────────────────────────────┐
│           物理网卡                  │
│  ┌──────┐  ┌────┐ ┌────┐ ┌────┐  │
│  │  PF  │  │ VF0│ │ VF1│ │ VF2│  │
│  │(宿主)│  │(VM1)│ │(VM2)│ │(VM3)│ │
│  └──────┘  └────┘ └────┘ └────┘  │
│         硬件包分流                  │
└────────────────────────────────────┘
```

硬件根据 MAC 地址或 VLAN 将数据包引导到正确的 VF，每个 VF 拥有专用的队列资源。没有软件交换开销。

### VF 创建与分配

```bash
# 检查 SR-IOV 能力
cat /sys/class/net/eth0/device/sriov_totalvfs    # 例如 63

# 创建 VF
echo 4 > /sys/class/net/eth0/device/sriov_numvfs

# 每个 VF 作为新的 PCI 设备出现
lspci | grep "Virtual Function"

# 将 VF 绑定到 vfio-pci 并分配给 VM（与任何设备相同的流程）
```

参见 [kvm-networking](../entities/kvm-networking.md) 了解使用 `<hostdev>` 和 `<interface type='hostdev'>` 进行 VF 分配的 libvirt XML 示例。

### SR-IOV 与完整设备直通对比

| 方面 | 完整设备直通 | SR-IOV VF 直通 |
|------|-----------|---------------|
| 每物理网卡的设备数 | 每网卡 1 个 VM | 每网卡多个 VM（最多 256 个 VF） |
| 性能 | 完整设备带宽 | 每 VF 带宽（硬件 QoS） |
| 宿主机驱动 | 已卸载（设备对宿主机不可用） | PF 仍在宿主机控制下 |
| 管理 | 设备对宿主机不可见 | 宿主机 PF 驱动管理 VF 创建 |

## 安全模型

VFIO 的安全性依赖三大支柱：

### 1. DMA 隔离

IOMMU 页表确保设备只能 DMA 到为其 VFIO 容器显式映射的内存。恶意设备（或被入侵的客户机驱动真实设备）无法读写宿主机内存、其他客户机的内存或 QEMU 的地址空间中超出映射区域的部分。

### 2. 中断隔离

中断重映射防止设备向任意 CPU 或向量注入中断。没有中断重映射的情况下，设备可以编写精心构造的 MSI 消息，针对任何 CPU 上的任意中断向量——可能向其他 VM 或虚拟机监控器注入中断。

### 3. 组级粒度

VFIO 强制要求 IOMMU 组中的所有设备必须由 VFIO（或已知安全的驱动）控制，然后组中的任何设备才能被分配。这防止了这样的场景：设备 A 被分配给 VM，而同组中的设备 B（仍在宿主机驱动下）可以作为代理，通过点对点 DMA 攻击 VM 的 IOMMU 域。

## 权衡与限制

**优势：**
- 接近原生的 I/O 性能：无模拟、无半虚拟化开销、数据路径中无虚拟机监控器
- 完整设备功能集暴露给客户机（硬件卸载、专有功能）
- 减少攻击面：I/O 路径中更少的 QEMU 设备模拟代码

**限制：**
- **不支持实时迁移**：设备状态位于硬件寄存器和内部 ASIC 状态中，无法捕获和传输。在迁移前必须关闭客户机或热拔出设备。这是相对于 virtio 的主要运维权衡。
- **设备共享受限**：没有 SR-IOV 时，一个物理设备服务一个 VM。即使有 SR-IOV，VF 数量也受硬件限制。
- **宿主机失去设备访问**：设备在分配给 VM 期间对宿主机操作系统不可用。
- **硬件依赖**：需要支持 IOMMU 的硬件（VT-d / AMD-Vi）、支持 IOMMU 的主板/BIOS，以及支持 ACS 的 PCIe 拓扑以实现细粒度隔离。
- **IOMMU 组约束**：多功能设备或非 ACS 交换机后的设备强制组级分配，可能需要直通比预期更多的设备。

## 参见

- [device-driver-model](../entities/device-driver-model.md)
- [kvm-networking](../entities/kvm-networking.md)
- [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md)
- [kvm-memory-virtualization](../entities/kvm-memory-virtualization.md)
- [kvm-performance-tuning](../entities/kvm-performance-tuning.md)
- [concept-hardware-virtualization](../concepts/concept-hardware-virtualization.md)
- [concept-virtio-data-plane](../concepts/concept-virtio-data-plane.md)
- [analysis-vfio-device-passthrough](analysis-vfio-device-passthrough.md) — English version
