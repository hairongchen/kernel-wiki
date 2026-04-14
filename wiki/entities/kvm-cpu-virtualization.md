---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [qemu-kvm-source-code-and-application]
tags: [kvm, cpu-virtualization, vmx, vmcs, vcpu]
---

# CPU Virtualization in QEMU/KVM

CPU virtualization is the foundation of QEMU/KVM's hypervisor architecture. It leverages Intel VT-x hardware extensions to run guest code directly on the physical CPU at near-native speed, with the hypervisor intervening only when the guest performs privileged operations that require emulation. This page covers the full stack from hardware VMX primitives through the KVM kernel module to the QEMU userspace VCPU lifecycle.

For the overall architecture, see [qemu-kvm-overview](qemu-kvm-overview.md).

## Intel VT-x and VMX Operations

Intel's Virtualization Technology for x86 (VT-x) introduces **VMX (Virtual Machine Extensions)**, which define two distinct CPU operating modes:

- **VMX root mode** -- the mode in which the hypervisor (KVM) runs. This is the privileged context that controls virtualization.
- **VMX non-root mode** -- the mode in which guest code runs. The guest operates as if it has full control of the hardware, but certain operations trigger exits back to root mode.

Both modes retain the full ring 0-3 privilege hierarchy. The guest kernel runs in non-root ring 0, and guest userspace runs in non-root ring 3. The hypervisor runs in root ring 0.

### Key VMX Instructions

| Instruction | Purpose |
|-------------|---------|
| `VMXON` | Enables VMX operation on the CPU. Enters VMX root mode. |
| `VMXOFF` | Disables VMX operation. Leaves VMX root mode entirely. |
| `VMLAUNCH` | Performs the first VM Entry into the guest using the current VMCS. |
| `VMRESUME` | Performs a subsequent VM Entry (after the guest has already been launched). |
| `VMEXIT` | Not an explicit instruction -- triggered by the hardware when the guest executes a sensitive operation. Transfers control from non-root to root mode. |

### VM Entry and VM Exit

A **VM Entry** (triggered by `VMLAUNCH` or `VMRESUME`) performs two atomic operations:

1. Saves the host processor state into the VMCS host-state area.
2. Loads guest processor state from the VMCS guest-state area.

A **VM Exit** reverses this:

1. Saves guest processor state into the VMCS guest-state area.
2. Loads host processor state from the VMCS host-state area.

This hardware-assisted context switch is what makes KVM fast -- there is no need for software-based register save/restore on every transition.

## VMCS (Virtual Machine Control Structure)

The **VMCS** is a per-VCPU data structure maintained in memory that the hardware reads and writes during VM Entry and VM Exit. It is the central mechanism for controlling guest execution and capturing exit information.

### VMCS Fields

The VMCS contains several groups of fields:

**Guest-state area** -- Loaded on VM Entry, saved on VM Exit:
- General-purpose registers, RIP, RSP, RFLAGS
- Segment registers and their descriptors (CS, DS, SS, ES, FS, GS, TR, LDTR)
- Control registers (CR0, CR3, CR4)
- Debug registers, MSRs (IA32_EFER, etc.)
- GDTR, IDTR base and limit

**Host-state area** -- Saved on VM Entry, loaded on VM Exit:
- Host CR0, CR3, CR4
- Host RSP, RIP (the entry point back into KVM after a VM Exit)
- Host segment selectors, GDTR, IDTR, TR
- Host MSRs

**VM-execution control fields** -- Configure what triggers a VM Exit:
- Pin-based controls (external interrupt exiting, NMI exiting)
- Processor-based controls (HLT exiting, I/O bitmap usage, MSR bitmaps, CR access exiting)
- Exception bitmap (which exceptions cause VM Exits)

**VM-exit control fields** -- Configure behavior on VM Exit (e.g., whether to save debug controls, host address-space size).

**VM-entry control fields** -- Configure behavior on VM Entry (e.g., whether to load debug controls, guest activity state, event injection).

**VM-exit information fields** -- Filled by the hardware on VM Exit with the exit reason, exit qualification, and other details needed by the hypervisor to handle the exit.

### VMCS Lifecycle in KVM

1. **`alloc_vmcs()`** -- Allocates a VMCS page from memory. Each VCPU gets its own VMCS.
2. **`vmptrld`** -- Binds a VMCS to the current physical CPU. Only one VMCS can be active per physical CPU at a time.
3. **`vmx_vcpu_setup()`** -- Initializes all VMCS fields: writes execution controls, sets up host-state area with current kernel state, configures guest-state area with initial values.

When a VCPU migrates to a different physical CPU, the VMCS must be unloaded from the old CPU (`vmclear`) and reloaded on the new one (`vmptrld`).

## QEMU-side VCPU Lifecycle

On the QEMU (userspace) side, a VCPU is represented as a QOM (QEMU Object Model) object. For x86 targets, the CPU type hierarchy is:

```
DeviceState → CPUState → X86CPU
```

The `X86CPU` structure contains `CPUX86State`, which holds the full architectural state: all general-purpose registers, segment registers, control registers, FPU/SSE/AVX state, MSRs, and APIC state.

### CPU Initialization

**`x86_cpu_initfn()`** -- Called when the CPU object is instantiated. Registers CPU feature properties (feature flags like SSE, AVX, etc.) as QOM properties so they can be configured from the command line or machine definition.

**`x86_cpu_realizefn()`** -- Called when the CPU object is "realized" (finalized). This function:
- Validates the requested CPU feature combination for consistency
- Applies feature filtering (e.g., removing features the host CPU does not support in KVM mode)
- Creates the VCPU thread that will drive this virtual CPU

### VCPU Threading Model

QEMU uses one host pthread per guest VCPU. The thread creation path is:

```
x86_cpu_realizefn()
  → qemu_kvm_start_vcpu()
    → pthread_create(qemu_kvm_cpu_thread_fn)
```

**`qemu_kvm_cpu_thread_fn()`** is the main function for each VCPU thread. Its structure is:

1. Call `kvm_init_vcpu()` to set up the VCPU with the KVM kernel module (issues `KVM_CREATE_VCPU` ioctl, mmaps the shared `kvm_run` structure).
2. Enter the main execution loop, repeatedly calling `kvm_cpu_exec()`.
3. The thread runs until the VM is shut down or the VCPU is hot-unplugged.

## kvm_cpu_exec() -- The VM Entry/Exit Loop

`kvm_cpu_exec()` is the core function that drives guest execution from userspace. Each call performs one round-trip into the guest and handles the resulting VM Exit.

### Execution Flow

```
kvm_cpu_exec():
  1. Synchronize pending state to KVM (interrupt injection, etc.)
  2. Call kvm_vcpu_ioctl(KVM_RUN)
     → enters the kernel, performs VM Entry
     → guest runs until a VM Exit occurs
     → KVM handles the exit if possible (many exits are handled in-kernel)
     → if KVM cannot handle it, returns to userspace
  3. Read run->exit_reason
  4. Switch on exit_reason:
     - KVM_EXIT_IO      → kvm_handle_io()
     - KVM_EXIT_MMIO    → address_space_rw()
     - KVM_EXIT_UNKNOWN → error handling
     - KVM_EXIT_SHUTDOWN → VM shutdown
     - ...other exit reasons...
  5. Return ret (0 = continue looping, nonzero = stop)
```

The main loop in `qemu_kvm_cpu_thread_fn()` continues calling `kvm_cpu_exec()` as long as it returns 0.

### Key Exit Handlers

**`kvm_handle_io()`** -- Handles port I/O exits. The guest performed an `IN` or `OUT` instruction to a port that is not handled in-kernel. QEMU dispatches this to the appropriate emulated device (e.g., a serial port, PCI config space access via port 0xCF8/0xCFC).

**`address_space_rw()`** -- Handles MMIO exits. The guest accessed a memory-mapped I/O region that is not backed by real RAM. QEMU looks up the corresponding `MemoryRegion` and dispatches the read or write to the emulated device. See [kvm-memory-virtualization](kvm-memory-virtualization.md) for details on how memory regions are managed.

## KVM-side VCPU Creation

When QEMU issues the `KVM_CREATE_VCPU` ioctl, the kernel-side creation path is:

```
KVM_CREATE_VCPU ioctl
  → kvm_vm_ioctl_create_vcpu()
    → vmx_create_vcpu()           [Intel-specific]
      1. Allocate vcpu_vmx from slab cache
      2. allocate_vpid()          [Virtual Processor ID for TLB tagging]
      3. kvm_vcpu_init()
         → kvm_arch_vcpu_init()
           - Create the virtual MMU (shadow page tables or EPT setup)
           - Create the virtual LAPIC (local APIC emulation)
      4. alloc_vmcs()             [Allocate the VMCS page]
      5. vmx_vcpu_load()
         → vmptrld              [Bind VMCS to current physical CPU]
      6. vmx_vcpu_setup()        [Initialize all VMCS fields]
    → kvm_arch_vcpu_setup()
      → kvm_vcpu_reset()        [Set initial register state]
      → kvm_mmu_setup()         [Initialize MMU context]
    → create_vcpu_fd()           [Return a file descriptor to userspace]
```

The returned file descriptor is what QEMU uses for all subsequent VCPU operations (`KVM_RUN`, `KVM_GET_REGS`, `KVM_SET_REGS`, etc.).

### vcpu_vmx Structure

On Intel systems, the KVM VCPU is represented by `struct vcpu_vmx`, which embeds the generic `struct kvm_vcpu`. Key fields include:

- The VMCS pointer (the allocated VMCS page)
- Host-side state that must be saved/restored beyond what the VMCS handles
- Nested VMX state (for running a hypervisor inside a guest)
- The VPID (Virtual Processor ID) assigned to this VCPU

## QEMU-KVM Shared Data: The kvm_run Structure

Communication between QEMU (userspace) and KVM (kernel) for each VCPU is done through a shared memory region: the `struct kvm_run`. This avoids expensive data copying on every VM Exit.

### Setup

1. QEMU calls `KVM_GET_VCPU_MMAP_SIZE` ioctl on the KVM device fd to learn the required mapping size.
2. QEMU mmaps the VCPU fd with `MAP_SHARED` at that size.
3. The resulting pointer is stored as `cpu->kvm_run`.

### Key Fields of kvm_run

| Field | Description |
|-------|-------------|
| `exit_reason` | Why the last `KVM_RUN` returned (e.g., `KVM_EXIT_IO`, `KVM_EXIT_MMIO`) |
| `io.direction` | For I/O exits: `KVM_EXIT_IO_IN` or `KVM_EXIT_IO_OUT` |
| `io.port` | For I/O exits: the port number accessed |
| `io.size` | For I/O exits: access width (1, 2, or 4 bytes) |
| `io.data_offset` | Offset into the kvm_run page where I/O data resides |
| `mmio.phys_addr` | For MMIO exits: the physical address accessed |
| `mmio.data` | For MMIO exits: the data read or to be written |
| `mmio.len` | For MMIO exits: access length |
| `mmio.is_write` | For MMIO exits: whether this was a write |

### Coalesced MMIO Ring

At an offset within the mmap'd region (beyond the `kvm_run` struct itself), KVM places a **coalesced MMIO ring buffer**. This optimization batches multiple MMIO writes to the same region (e.g., consecutive writes to a device FIFO) so that KVM can record them without exiting to userspace for each one. QEMU drains this ring when it next processes exits.

## CPU Reset

When a VCPU is reset (at creation time or on triple fault), `x86_cpu_reset()` sets the initial architectural state to match the Intel SDM-defined reset state:

- **CS:EIP** = `0xF000:0xFFF0` -- the x86 reset vector. The CPU begins execution at physical address `0xFFFF0`, which is typically mapped to the BIOS/firmware entry point.
- **EFLAGS** = `0x2` (bit 1 is always set, all other flags cleared).
- **Segment limits** = `0xFFFF` (64 KB) for all segment registers.
- **CR0** = real mode (PE bit clear).

### BSP vs AP Startup

The **first VCPU** (VCPU 0) is designated as the **Bootstrap Processor (BSP)**. It begins executing immediately from the reset vector. All other VCPUs are **Application Processors (APs)** and start in a halted state. They remain halted until the BSP sends INIT and SIPI (Startup IPI) interrupts to wake them, following the MP initialization protocol. See [kvm-interrupt-virtualization](kvm-interrupt-virtualization.md) for details on interrupt delivery.

## Relationship Between Components

The following summarizes how the pieces fit together in a typical VCPU execution cycle:

1. QEMU creates the VCPU (QOM object + KVM ioctl), gets back a VCPU fd.
2. QEMU mmaps the VCPU fd to get the shared `kvm_run` structure.
3. The VCPU thread enters `kvm_cpu_exec()`, which calls `KVM_RUN`.
4. KVM loads the VMCS, executes `VMLAUNCH`/`VMRESUME`, and the guest runs in VMX non-root mode.
5. A VM Exit occurs (I/O, MMIO, HLT, interrupt window, etc.).
6. KVM handles the exit in-kernel if possible (e.g., EPT violations for page faults, APIC accesses).
7. If the exit requires device emulation, KVM fills `kvm_run` and returns to userspace.
8. QEMU reads `exit_reason`, dispatches to the appropriate handler, and loops back to step 3.

This cycle repeats millions of times per second during normal guest operation. The efficiency of the system depends on minimizing the number of exits that reach userspace -- most exits are handled entirely within KVM in the kernel.

## See also

- [qemu-kvm-overview](qemu-kvm-overview.md) -- Overall QEMU/KVM architecture
- [kvm-memory-virtualization](kvm-memory-virtualization.md) -- EPT, shadow page tables, memory region management
- [kvm-interrupt-virtualization](kvm-interrupt-virtualization.md) -- APIC virtualization, interrupt injection, SIPI
- [kvm-pmu-virtualization](kvm-pmu-virtualization.md) -- PMU virtualization: emulated and mediated modes, counter management
- [qemu-machine-emulation](qemu-machine-emulation.md) -- Machine types, device emulation, QOM
