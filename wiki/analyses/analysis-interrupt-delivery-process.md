---
type: analysis
created: 2026-04-10
updated: 2026-04-10
sources: [understanding-the-linux-kernel, professional-linux-kernel-architecture, qemu-kvm-source-code-and-application]
tags: [interrupts, apic, ipi, interrupt-delivery, smp, halt, idle, posted-interrupts]
---

# Interrupt Delivery Process

This analysis traces the complete lifecycle of an interrupt from the moment a hardware device (or software source) asserts it to the moment the target CPU begins executing the handler. The two critical scenarios — target CPU is **actively running** vs. **halted/idle** — are examined in detail, as the hardware and software paths diverge significantly.

## 1. Interrupt Sources and Routing Overview

Before delivery can happen, the interrupt must travel from its source to a specific CPU. The path depends on the interrupt controller topology:

```
                        +-----------+
  Device IRQ pin ------>| I/O APIC  |---+
                        +-----------+   |    +------------+
                                        +--->| Local APIC |---> CPU core
  MSI/MSI-X write --------------------------| (per-CPU)  |
                                        +--->|            |
  IPI (from another CPU) --------------+    +------------+
                                             |
  Local sources (timer, LINT, error) --------+
```

### I/O APIC Routing Decision

When a device asserts an IRQ line connected to the I/O APIC, the controller consults its **redirection table** entry for that pin. The entry specifies:

| Field | Role in delivery |
|-------|-----------------|
| `destination` | Target APIC ID(s) — physical or logical |
| `dest_mode` | **Physical**: single APIC ID. **Logical**: bitmask matched against LDR/DFR |
| `delivery_mode` | Fixed, Lowest Priority, NMI, SMI, INIT, ExtINT |
| `trigger_mode` | Edge or level — affects acknowledgment protocol |
| `vector` | The IDT vector number (32-255) the target CPU will use |
| `mask` | If set, interrupt is suppressed |

For **Lowest Priority** delivery mode, the I/O APIC (or on newer hardware, the CPU arbitration logic) selects the CPU with the lowest Task Priority Register (TPR) value — the one currently doing the least important interruptible work.

### MSI Bypass

MSI/MSI-X interrupts encode the destination APIC ID and vector directly in a memory write transaction. The write targets address `0xFEE0_0000` (the LAPIC MMIO range), and the chipset delivers it directly to the specified Local APIC, bypassing the I/O APIC entirely.

## 2. Local APIC Interrupt Acceptance

Regardless of the source (I/O APIC, MSI, IPI, or local), the interrupt arrives at the target CPU's **Local APIC**. The acceptance logic proceeds through several stages:

### Stage 1: IRR — Recording the Request

The Local APIC has a 256-bit **Interrupt Request Register (IRR)**, one bit per vector. When an interrupt message arrives, the APIC sets the corresponding bit in the IRR. For edge-triggered interrupts, the bit is set on the rising edge; for level-triggered, it remains set as long as the source asserts.

### Stage 2: Priority Filtering (TPR/PPR)

The APIC compares the priority of the highest pending vector in the IRR against the **Processor Priority Register (PPR)**:

```
PPR = max(TPR, highest-priority vector in ISR)
```

- **TPR** (Task Priority Register): software-controlled. The OS sets this to mask interrupts below a certain priority class. The upper 4 bits define a priority class (0-15); vectors within a class share the same priority level (each class spans 16 vectors).
- **ISR** (In-Service Register): tracks the vector currently being serviced.

If the pending vector's priority class is **higher** than the PPR's priority class, the interrupt is eligible for delivery. Otherwise, it remains pending in the IRR until the PPR drops (e.g., after an EOI).

### Stage 3: Delivery to the CPU Core

Once a vector passes the priority filter, the APIC asserts the **INTR pin** (or its logical equivalent on the internal bus) to the CPU core. What happens next depends on whether the CPU is running or halted.

## 3. Scenario A: Target CPU Is Running

This is the most common scenario during normal system operation. The CPU is executing instructions (kernel code, user-space code, or another interrupt handler).

### 3A.1 — The INTR Signal and EFLAGS.IF

The CPU checks the INTR signal at **instruction boundaries** (between completing one instruction and fetching the next). Two conditions must be met:

1. **EFLAGS.IF = 1** (interrupts enabled). If IF is cleared (e.g., inside a `cli`/`sti` critical section or during another interrupt handler's top half), the CPU ignores INTR. The interrupt remains pending in the LAPIC's IRR until IF is set again.
2. No higher-priority event is pending (NMI, SMI, or debug exception take precedence).

```
Instruction N completes
    |
    v
[Check INTR && IF==1?]
    |           |
   YES          NO ---> continue to instruction N+1
    |                   (interrupt stays in IRR)
    v
Begin interrupt acceptance
```

### 3A.2 — Hardware Interrupt Acceptance Sequence

When the CPU accepts the interrupt, the following hardware sequence occurs **atomically** (no software involvement):

1. **Acknowledge the LAPIC**: The CPU reads the interrupt vector from the LAPIC (interrupt acknowledge cycle). The LAPIC:
   - Clears the corresponding bit in the **IRR**.
   - Sets the corresponding bit in the **ISR** (In-Service Register).
   - If there are additional pending interrupts with higher priority, keeps INTR asserted.

2. **Save interrupted context onto the stack**:
   - Push `SS` and `ESP` (if privilege level changes, i.e., user→kernel transition)
   - Push `EFLAGS`
   - Push `CS`
   - Push `EIP` (the address of the **next** instruction — the one that would have executed)
   - If the IDT entry is an **interrupt gate**: clear `EFLAGS.IF` (disable further maskable interrupts)
   - If the IDT entry is a **trap gate**: leave `EFLAGS.IF` unchanged

3. **Look up the IDT**: Use the vector as an index into the 256-entry IDT to find the gate descriptor.

4. **Load the handler**: Set `CS:EIP` to the handler address from the gate descriptor. If a privilege transition occurred, switch to the kernel stack (from the TSS).

### 3A.3 — Software Handling Path (Linux)

The IDT entries for hardware interrupts (vectors 32-255) point to assembly stubs that converge on the common path:

```
Entry stub (vector-specific)
  |
  v
common_interrupt:
  SAVE_ALL                    /* push all general-purpose registers */
  |
  v
do_IRQ(struct pt_regs *regs):
  irq_enter()                 /* preempt_count += HARDIRQ_OFFSET */
  |
  v
  irq_desc[irq].handle_irq() /* genirq flow handler */
    |
    +-- handle_level_irq()  or  handle_edge_irq()  or  handle_fasteoi_irq()
    |     |
    |     v
    |   chip->ack() / chip->mask_ack()
    |   walk irqaction chain: call each handler(irq, dev_id)
    |   chip->unmask() / chip->eoi()
    |
  irq_exit()                  /* preempt_count -= HARDIRQ_OFFSET */
    |                         /* if softirqs pending: do_softirq() */
    v
ret_from_intr:
  if returning to user mode:
    check TIF_NEED_RESCHED  --> schedule()
    check TIF_SIGPENDING    --> do_signal()
  if returning to kernel mode:
    check preempt_count == 0 && TIF_NEED_RESCHED --> preempt_schedule_irq()
  RESTORE_ALL
  iret
```

### 3A.4 — Latency Characteristics (Running CPU)

| Phase | Approximate latency | Notes |
|-------|-------------------|-------|
| I/O APIC → LAPIC message | ~1 us | Inter-chip bus message |
| LAPIC arbitration + IRR set | ~100 ns | On-die logic |
| Wait for instruction boundary | 0 — ~100 ns | Depends on current instruction |
| IF check + accept + context save | ~50-100 ns | Hardware microcode |
| IDT lookup + jump to handler | ~10-20 ns | In L1 cache |
| SAVE_ALL + do_IRQ entry | ~100-200 ns | Software overhead |
| **Total to handler entry** | **~1-2 us** | Typical, varies with load |

### 3A.5 — The IF=0 Case (Interrupts Disabled)

If the CPU is running with `IF=0` — inside a `spin_lock_irqsave()` critical section, another interrupt's top half, or an explicit `local_irq_disable()` region — the interrupt remains pending in the LAPIC IRR. No CPU cycles are spent on it.

When interrupts are re-enabled (`sti`, `local_irq_enable()`, `spin_unlock_irqrestore()`), the CPU immediately checks INTR at the next instruction boundary and begins the acceptance sequence. The `sti` instruction has a one-instruction delay: the CPU will not recognize INTR until after the instruction following `sti` completes (the "STI shadow"). This allows code like:

```asm
sti
hlt     ; atomically enters halt with interrupts enabled
```

This is critical for the halted-CPU scenario described next.

## 4. Scenario B: Target CPU Is Halted (Idle)

When a CPU has no runnable tasks, the Linux idle loop puts it into a low-power halt state. This is the common state for lightly loaded systems — on a 64-core machine running a single-threaded workload, 63 cores are typically halted.

### 4B.1 — The Idle Loop and HLT

The Linux idle loop (simplified) looks like:

```c
void cpu_idle(void) {
    while (1) {
        while (!need_resched()) {
            /* enter C-state (halt or deeper) */
            local_irq_disable();
            if (!need_resched())
                safe_halt();    /* sti; hlt — atomic */
            else
                local_irq_enable();
        }
        schedule();   /* pick next task */
    }
}
```

The `safe_halt()` macro expands to the `sti; hlt` instruction sequence. The x86 architecture guarantees that the **STI shadow** prevents interrupt recognition between `sti` and `hlt`, so the CPU atomically transitions to:

- **IF = 1** (interrupts enabled)
- **Halted state** (no instruction fetch, minimal power)

This atomicity is essential. Without it, a race exists: an interrupt arriving between `sti` and `hlt` would be handled (clearing the need for the wakeup), and then `hlt` would execute, potentially halting the CPU indefinitely until the next interrupt.

#### Deeper C-States (ACPI)

Modern processors support C-states deeper than HLT (C1):
- **C1 (HLT)**: Clock gated. Wakes in ~1 us.
- **C1E**: Enhanced halt. Wakes in ~10 us.
- **C3 (Sleep)**: L1/L2 cache may be flushed. Wakes in ~100 us.
- **C6 (Deep Power Down)**: Core voltage reduced. Wakes in ~200+ us.

The `cpuidle` governor chooses the appropriate C-state based on expected idle duration. Deeper states save more power but have higher **exit latency**, directly impacting interrupt response time.

### 4B.2 — Wakeup Sequence

When an interrupt arrives at a halted CPU:

```
LAPIC receives interrupt message
    |
    v
IRR bit set, priority check passes
    |
    v
LAPIC asserts INTR to CPU core
    |
    v
[CPU is halted — special wakeup path]
    |
    v
CPU exits halt state (C-state exit):
  - restore clocks (C1: ~1 us)
  - restore caches (C3: ~100 us)
  - restore voltage (C6: ~200 us)
    |
    v
Resume at instruction AFTER hlt
    |
    v
CPU checks INTR (IF is already 1 from sti before hlt)
    |
    v
Normal interrupt acceptance sequence (same as 3A.2)
  - acknowledge LAPIC, IRR→ISR
  - save context, look up IDT
  - jump to handler
```

The key insight: **HLT is not a barrier to interrupt delivery** — it is specifically designed to be broken by interrupts (and NMIs). The `HLT` instruction is defined to resume at the next instruction when INTR (or NMI) is asserted, provided IF=1 for maskable interrupts.

### 4B.3 — Latency Characteristics (Halted CPU)

| Phase | C1 (HLT) | C3 (Sleep) | C6 (Deep) |
|-------|-----------|------------|-----------|
| I/O APIC → LAPIC | ~1 us | ~1 us | ~1 us |
| C-state exit | ~1 us | ~50-100 us | ~100-200+ us |
| Post-HLT interrupt accept | ~100 ns | ~100 ns | ~100 ns |
| Software handler entry | ~300 ns | ~300 ns | ~300 ns |
| **Total** | **~2-3 us** | **~50-100 us** | **~100-200+ us** |

The C-state exit latency **dominates** for deep sleep states. This is the primary reason latency-sensitive workloads (networking, real-time) configure `idle=poll` (never halt) or limit C-state depth via `/dev/cpu_dma_latency` or kernel parameters (`processor.max_cstate`).

### 4B.4 — The Idle Loop After Wakeup

After the interrupt handler returns via `iret`, execution resumes in the idle loop at the instruction after `hlt`. The loop re-checks `need_resched()`:

- If the interrupt handler marked a task as runnable (e.g., a network packet woke a blocked process), `TIF_NEED_RESCHED` is set, and `schedule()` picks up the new task.
- If no rescheduling is needed, the CPU re-enters halt.

## 5. Inter-Processor Interrupts (IPIs)

IPIs are the mechanism by which one CPU interrupts another. They are critical for:

| Purpose | Vector/Function | Target state matters? |
|---------|----------------|----------------------|
| **TLB shootdown** | `INVALIDATE_TLB_VECTOR` | Yes — halted CPU has no valid TLB entries, may skip |
| **Reschedule** | `RESCHEDULE_VECTOR` / `smp_send_reschedule()` | Yes — must wake halted CPU |
| **Function call** | `CALL_FUNCTION_VECTOR` / `smp_call_function()` | Yes — must wake to execute |
| **Stop** | `smp_send_stop()` | Yes — must wake to stop |

### Sending an IPI

An IPI is sent by writing to the sender's Local APIC **Interrupt Command Register (ICR)**:

```c
/* Simplified: send IPI to a specific CPU */
apic_write(APIC_ICR2, target_apic_id << 24);  /* destination */
apic_write(APIC_ICR, vector | APIC_DEST_PHYSICAL | APIC_DM_FIXED);
```

The LAPIC sends the IPI message over the system bus (or APIC bus on older systems) to the target LAPIC. The delivery follows the same IRR → priority check → INTR assertion path as any other interrupt.

### IPI to a Running CPU

The target CPU is interrupted at the next instruction boundary (assuming IF=1). The IPI handler runs, performs the requested action (flush TLB, set `TIF_NEED_RESCHED`, execute a function), and returns. The interrupted code resumes transparently.

### IPI to a Halted CPU

The IPI wakes the target CPU from its halt state. After the C-state exit, the IPI handler executes. This is how the scheduler wakes an idle CPU to run a newly runnable task:

```c
/* kernel/sched/core.c — simplified */
void wake_up_process(struct task_struct *p) {
    /* ... select target CPU ... */
    if (task_cpu(p) != smp_processor_id()) {
        /* target is on another CPU — may be halted */
        smp_send_reschedule(task_cpu(p));  /* IPI */
    }
}
```

The reschedule IPI handler simply sets `TIF_NEED_RESCHED`. When the handler returns to the idle loop, `need_resched()` returns true, and `schedule()` picks up the newly woken task.

## 6. Edge-Triggered vs. Level-Triggered Delivery

The trigger mode affects the delivery contract between source and LAPIC:

### Edge-Triggered

```
Signal: ___/‾‾‾‾\___

IRR set on rising edge (0→1 transition)
Once set in IRR, the source can deassert
Second edge while IRR bit is set: LOST (no queuing in APIC)
```

- Used by: MSI/MSI-X, legacy ISA devices (typically)
- Risk: **lost interrupts** if the device re-asserts before the handler clears the source. Drivers must check device status registers and re-check after handling.
- genirq flow: `handle_edge_irq()` — ack first, then handle. If a new edge arrives during handling, the handler re-runs.

### Level-Triggered

```
Signal: ___/‾‾‾‾‾‾‾‾‾‾‾\___
              ^            ^
              IRR set      IRR cleared after EOI +
                           device deasserts

```

- Used by: PCI conventional devices (shared lines)
- The LAPIC's IRR bit remains set as long as the line is asserted
- After EOI, if the line is still asserted (device not yet serviced), the IRR bit is immediately re-set, triggering another interrupt
- genirq flow: `handle_level_irq()` — mask+ack, then handle, then unmask. Masking prevents interrupt storms during handler execution.

### Impact on Halted CPUs

Both trigger modes wake a halted CPU identically — the LAPIC asserts INTR regardless. The difference matters only for the **acknowledgment protocol** after the CPU wakes up and handles the interrupt.

## 7. The Complete Picture — Side by Side

```
                    RUNNING CPU                          HALTED CPU
                    (IF=1 assumed)                       (in HLT, IF=1)
                         |                                    |
  Interrupt arrives      |                                    |
  at Local APIC          |                                    |
         |               |                                    |
         v               v                                    v
    Set IRR bit     Set IRR bit                          Set IRR bit
         |               |                                    |
         v               v                                    v
    Priority >PPR?  Check INTR at                        Assert INTR
    Yes: assert     instruction                          (CPU is halted)
    INTR            boundary                                  |
         |               |                                    v
         |               v                               C-state exit
         |          IF==1?                               (1us — 200+us)
         |          Yes: accept                               |
         |               |                                    v
         |               v                               Resume after HLT
         |          IRR→ISR                                   |
         |          Push context                              v
         |          Lookup IDT                           Check INTR (IF=1)
         |          Jump to handler                      Accept interrupt
         |               |                                    |
         |               v                                    v
         |          do_IRQ()                             Same as running:
         |          irqaction chain                      IRR→ISR, push ctx,
         |          irq_exit()                           IDT, handler
         |          ret_from_intr                             |
         |               |                                    v
         |               v                              do_IRQ(), etc.
         |          Resume interrupted                       |
         |          code                                     v
         |                                              ret_from_intr
         |                                              → idle loop
         |                                              → need_resched()?
         |                                              → schedule() or
         |                                                re-enter HLT
```

## 8. Practical Implications

### Latency Tuning

| Knob | Effect |
|------|--------|
| `idle=poll` | Never halt — lowest latency, highest power | 
| `processor.max_cstate=1` | Only use C1 (HLT) — ~1us exit | 
| `/dev/cpu_dma_latency` | PM QoS constraint — cpuidle governor respects it | 
| `irqbalance` daemon | Distributes IRQs across CPUs — avoids waking wrong CPU | 
| `/proc/irq/N/smp_affinity` | Pin IRQs to specific CPUs — avoid halted targets | 
| `isolcpus` + `irqaffinity` | Dedicate CPUs to specific workloads | 

### The Wake-then-Schedule Pattern

A common pattern in the kernel: device interrupt arrives → handler processes data → wakes a blocked process → if that process's CPU is halted, send a reschedule IPI. This involves **two** interrupt deliveries:

1. Device interrupt → CPU A (wherever the IRQ is routed)
2. Reschedule IPI → CPU B (where the target task last ran, possibly halted)

Intelligent IRQ affinity can reduce this to one delivery by routing the device interrupt to the same CPU where the waiting process sleeps.

### KVM/Virtualization Considerations

In a virtualized environment, interrupt delivery to a guest VCPU adds another layer. See [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md) for the posted-interrupt mechanism, which handles both scenarios:

- **VCPU running in guest mode**: Posted-interrupt notification IPI causes hardware to merge `pir[]` bits into the virtual VIRR — no VM Exit.
- **VCPU not running (scheduled out)**: Pending interrupts are synced from `pir[]` to IRR at the next VM Entry via `vmx_sync_pir_to_irr()`.

## See also

- [interrupt-handling](../entities/interrupt-handling.md) — IDT, PIC/APIC architecture, softirqs, tasklets, work queues
- [concept-napi-interrupt-mitigation](../concepts/concept-napi-interrupt-mitigation.md) — Interrupt-to-polling transition for networking
- [kvm-interrupt-virtualization](../entities/kvm-interrupt-virtualization.md) — Virtual interrupt delivery with APICv/posted interrupts
- [concept-kernel-synchronization](../concepts/concept-kernel-synchronization.md) — Synchronization primitives affected by interrupt context
- [process-scheduler](../entities/process-scheduler.md) — Scheduler interaction with interrupt-driven wakeups
- [timing-subsystem](../entities/timing-subsystem.md) — Timer interrupts as a special case of local APIC delivery
- [analysis-interrupt-delivery-process-zh](analysis-interrupt-delivery-process-zh.md) — Chinese version / 中文版
