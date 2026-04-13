---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [qemu-kvm-source-code-and-application]
tags: [qemu, kvm, virtualization, qom, event-loop, qmp]
---

# QEMU/KVM Architecture Overview

QEMU (Quick Emulator) and KVM (Kernel-based Virtual Machine) form the dominant open-source virtualization stack on Linux. QEMU provides full-system emulation in userspace, while KVM is a kernel module that leverages hardware virtualization extensions (Intel VT-x / AMD-V) to run guest code at near-native speed. This page covers the architecture as described in the QEMU/KVM source code analysis by Li Qiang (2021), based on QEMU 2.8.1 and Linux kernel 4.4.161.

## QEMU/KVM Execution Architecture

### Three Execution Modes

A QEMU/KVM virtual machine operates across three distinct execution modes:

1. **Guest mode (VMX non-root)**: The guest OS runs directly on the physical CPU in VMX non-root operation. The hardware enforces isolation — the guest believes it has full control of the CPU, but privileged operations or certain events trigger a VM-Exit.

2. **Kernel mode (KVM — VM-Exit handler)**: When a VM-Exit occurs, the CPU transitions to VMX root operation and control passes to the KVM module in the host kernel. KVM handles the exit reason (e.g., I/O port access, MSR read/write, page fault in EPT/NPT) and either resolves it directly in kernel space or returns to userspace if QEMU assistance is needed.

3. **Userspace mode (QEMU)**: QEMU handles complex device emulation, disk and network I/O, and management operations that KVM cannot resolve in-kernel. After processing, QEMU issues another `ioctl(KVM_RUN)` to re-enter the guest.

### The /dev/kvm Interface

KVM exposes its functionality through a layered ioctl interface on `/dev/kvm`:

- **System level** (`/dev/kvm` fd): Query capabilities (`KVM_GET_API_VERSION`, `KVM_CHECK_EXTENSION`), create VMs (`KVM_CREATE_VM`).
- **VM level** (VM fd): Configure VM-wide state — create VCPUs (`KVM_CREATE_VCPU`), set memory regions (`KVM_SET_USER_MEMORY_REGION`), create interrupt controllers (`KVM_CREATE_IRQCHIP`).
- **VCPU level** (VCPU fd): Run the guest (`KVM_RUN`), get/set registers (`KVM_GET_REGS`, `KVM_SET_REGS`), inject interrupts.

### VM Entry/Exit Loop

The core execution cycle is:

```
QEMU (userspace)
  │
  ├─ ioctl(vcpu_fd, KVM_RUN)
  │       │
  │       ▼
  │   KVM (kernel) ──── VM-Entry ────► Guest (VMX non-root)
  │       ▲                                    │
  │       │                              VM-Exit (privileged op,
  │       │                               I/O, interrupt, etc.)
  │       │                                    │
  │       └──── handle exit reason ◄───────────┘
  │       │
  │       ├─ [handled in kernel] → re-enter guest
  │       └─ [needs userspace]  → return to QEMU
  │
  ├─ process exit reason (device emulation, I/O)
  └─ loop back to KVM_RUN
```

QEMU allocates guest RAM as host userspace memory (typically via `mmap`). KVM maps this into the guest physical address space using `KVM_SET_USER_MEMORY_REGION`, and the hardware MMU virtualization (EPT/NPT) translates guest-physical to host-physical addresses. See [kvm-memory-virtualization](kvm-memory-virtualization.md) for details.

Each VCPU is backed by a dedicated QEMU thread. See [kvm-cpu-virtualization](kvm-cpu-virtualization.md) for the CPU virtualization mechanisms, and [kvm-interrupt-virtualization](kvm-interrupt-virtualization.md) for interrupt delivery.

## QOM — QEMU Object Model

QOM is QEMU's C-language object system providing type hierarchies, polymorphism, and introspection. It underpins the entire device model.

### Type Registration and Initialization

The lifecycle has three stages:

1. **TypeInfo → TypeImpl**: The developer writes a `TypeInfo` struct (name, parent, class_init, instance_init, class/instance sizes) and registers it via `type_register_static()`. Internally QEMU creates a `TypeImpl` stored in the global `type_table` hash table, keyed by type name.

2. **type_initialize() — class init (lazy)**: The first time a type is needed, `type_initialize()` is called. It is recursive — it initializes the parent class first, copies the parent's `ObjectClass` data into the child's class struct (inheriting vtable entries), then calls the child's `class_init()` to override methods. This happens once per type.

3. **object_new() — instance creation**: Allocates memory for the `Object` (sized by `instance_size`), initializes the base `Object` fields (type pointer, reference count), then walks the type hierarchy calling each level's `instance_init()` from parent to child.

### C-Language Polymorphism

QOM achieves polymorphism through struct embedding: a child type's instance struct embeds the parent struct as its **first member**. This means a pointer to the child can be safely cast to a pointer to the parent (and vice versa with appropriate checks).

```c
struct PCIDevice {
    DeviceState parent_obj;   /* must be first member */
    /* PCI-specific fields ... */
};
```

Type-safe casting is done via macros:

- `OBJECT_CHECK(type, obj, TYPE_NAME)` — casts an `Object*` to a specific instance type, with runtime type verification in debug builds.
- `OBJECT_CLASS_CHECK(class_type, klass, TYPE_NAME)` — casts an `ObjectClass*` to a specific class type.
- `OBJECT_GET_CLASS(obj)` — retrieves the class pointer from an object instance.

### Core Type Hierarchy

**Instance hierarchy:**
```
Object
  └── DeviceState
        ├── PCIDevice
        ├── ISADevice
        └── SysBusDevice
```

**Class hierarchy:**
```
ObjectClass
  └── DeviceClass
        ├── PCIDeviceClass
        ├── ISADeviceClass
        └── SysBusDeviceClass
```

`ObjectClass` carries the base vtable (type metadata, cast validation). `DeviceClass` adds device-specific virtual methods (`realize`, `unrealize`, `reset`). Concrete device classes override these to implement specific hardware behavior.

### Properties

Objects carry key-value properties used for configuration and introspection:

- **Child properties**: Represent ownership. Adding a child property increments the child's reference count; removing it decrements. This forms an object composition tree.
- **Link properties**: Non-owning references to other objects. They do not affect reference counts. Used for cross-references (e.g., a NIC linking to its backend network device).

Properties are the primary mechanism for configuring devices from the command line (`-device virtio-net-pci,netdev=net0`) and via QMP.

## QDev — Device Model

QDev is the device abstraction layer built on top of QOM, providing a uniform lifecycle for all emulated devices.

### Key Structures

- **DeviceClass**: The class-level structure. Key fields include `realize` and `unrealize` callbacks (virtual methods for device initialization and teardown), `props` (property definitions), `vmsd` (VMStateDescription for [vm-live-migration](vm-live-migration.md)), `reset`, and `hotplug`/`unplug` handlers.

- **DeviceState**: The instance-level structure. Contains the `realized` flag (whether the device has been fully initialized), a string `id` (user-assigned name), a pointer to the `parent_bus`, and a list of child buses. The `realized` flag is itself a property, which triggers the realize cascade when set to `true`.

- **BusState**: Represents a bus (PCI bus, ISA bus, system bus). Holds a pointer to the parent device that owns the bus and a list of child devices attached to it. Buses enforce topology — devices attach to buses, which attach to controller devices.

### Device Lifecycle

```
qdev_create(bus, type_name)
    → object_new(type_name)       // allocate + instance_init chain
    → set parent bus
    ▼
set properties                     // command-line or QMP
    ▼
device_set_realized(dev, true)
    → DeviceClass.realize(dev)    // type-specific init (register I/O, PCI BARs, etc.)
    → realize child buses
```

The `realize` callback is where the device performs substantive initialization: registering MMIO/PIO regions, connecting interrupt lines, initializing internal state. `unrealize` reverses this during hot-unplug or shutdown. See [qemu-machine-emulation](qemu-machine-emulation.md) for how devices are composed into a complete machine.

## Event Loop and Threading

### GLib Main Loop

QEMU's main thread runs a GLib-based event loop (`GMainLoop`). The loop dispatches:

- **File descriptor events**: Monitoring sockets, character devices, block device completions.
- **Timers**: Both real-time and virtual-time clocks.
- **Idle callbacks**: Deferred work when no other events are pending.

### AioContext

`AioContext` is QEMU's abstraction for asynchronous I/O contexts. Each `AioContext` manages its own set of:

- **fd handlers**: Callbacks triggered when file descriptors become readable/writable.
- **Bottom halves (BHs)**: Deferred callbacks scheduled to run on the next iteration of the event loop. BHs are useful for breaking out of a deep call stack or deferring work from an interrupt-like context.
- **Timers**: Time-based callbacks within the async context.

The main thread has a default `AioContext`. Block layer I/O threads can have their own `AioContext` instances, enabling parallel I/O processing.

### Threading Model

- **Main thread**: Runs the GLib main loop, handles QMP commands, device emulation, and management tasks.
- **VCPU threads**: One per virtual CPU. Each thread runs the `KVM_RUN` loop and handles VM-Exits. When a VM-Exit requires QEMU device emulation, the VCPU thread acquires the BQL and performs the emulation directly.
- **I/O threads** (optional): Dedicated threads for block I/O, each with their own `AioContext`.

### Big QEMU Lock (BQL)

The BQL (historically called the "global mutex" or "iothread lock") serializes access to shared device state. The main thread typically holds the BQL. VCPU threads acquire it when they need to access shared state (e.g., during device emulation on a VM-Exit). This coarse-grained locking simplifies correctness but can become a bottleneck; ongoing work in QEMU aims to reduce BQL scope with finer-grained locking.

## HMP and QMP — Monitor Protocols

QEMU provides two monitor interfaces for runtime control and introspection.

### HMP — Human Monitor Protocol

HMP is a text-based interactive console designed for human operators. Commands like `info registers`, `migrate`, and `device_add` are typed directly. HMP is convenient for debugging but difficult to parse programmatically.

### QMP — QEMU Machine Protocol

QMP is a JSON-based protocol designed for machine consumption (used by libvirt, virsh, and other management tools). Communication follows a request-response pattern over a Unix or TCP socket.

**Dispatch flow:**

```
Client sends JSON: {"execute": "query-status"}
       │
       ▼
  parse JSON (QObject layer)
       │
       ▼
  extract "execute" field
       │
       ▼
  qmp_find_command("query-status")
       │
       ▼
  marshaller function (auto-generated)
       │ - deserializes arguments from JSON → C structs
       │ - calls implementation: qmp_query_status()
       │ - serializes return value from C structs → JSON
       ▼
  send JSON response to client
```

### QAPI Schema and Code Generation

The QAPI (QEMU API) schema (`qapi-schema.json`) defines all QMP commands, events, and data types in a declarative format. A Python-based code generator processes the schema and produces:

- **C type definitions**: Structs and enums matching the schema types.
- **Visitor functions**: Implement the visitor pattern to walk data structures:
  - `QmpInputVisitor`: Deserializes JSON (QObject) → C struct. Walks the JSON tree and populates struct fields.
  - `QmpOutputVisitor`: Serializes C struct → JSON (QObject). Walks the C struct and builds the JSON representation.
- **Marshaller functions**: Glue code that ties a QMP command name to its implementation, using visitors for argument deserialization and result serialization.

This approach ensures type safety between the JSON wire format and C implementation, and eliminates hand-written boilerplate for each command.

## See also

- [kvm-cpu-virtualization](kvm-cpu-virtualization.md)
- [kvm-memory-virtualization](kvm-memory-virtualization.md)
- [kvm-interrupt-virtualization](kvm-interrupt-virtualization.md)
- [virtio-framework](virtio-framework.md)
- [vm-live-migration](vm-live-migration.md)
- [qemu-machine-emulation](qemu-machine-emulation.md)
