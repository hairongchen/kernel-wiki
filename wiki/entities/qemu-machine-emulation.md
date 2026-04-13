---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [qemu-kvm-source-code-and-application]
tags: [qemu, i440fx, piix3, seabios, acpi, vga, firmware]
---

# QEMU Machine and Platform Emulation

QEMU emulates a complete PC platform so that a guest OS encounters familiar hardware. The default x86 machine type models an **Intel 440FX**-based motherboard with a PIIX3 southbridge, SeaBIOS firmware, ACPI tables, interrupt controllers, timers, VGA, and serial ports. This page describes how QEMU assembles and initializes that platform (based on QEMU 2.8.1, kernel 4.4.161, from the analysis by Li Qiang, 2021).

See [qemu-kvm-overview](qemu-kvm-overview.md) for the high-level QEMU/KVM architecture.

## Intel 440FX Chipset

The emulated PC follows the classic two-chip Intel 440FX design:

**Northbridge (i440FX — PCI host bridge + memory controller):** Connects the CPU(s) to the PCI bus and to main memory. In QEMU the northbridge sits at PCI device 00:00.0 and owns the `PCIBus` object through which all PCI devices are enumerated.

**Southbridge (PIIX3 — PCI-to-ISA bridge):** Sits at PCI device 00:01.0 and aggregates legacy and low-speed I/O:

- **IDE controller** (PIIX3-IDE) — two channels, up to four disks.
- **USB controller** (UHCI) — USB 1.1 host controller.
- **Power management / SMBus** (PIIX4-PM) — ACPI control block, SCI interrupt, SMBus for SPD and sensor access.
- **Legacy devices** bridged onto the ISA bus: DMA (i8237), dual PIC (i8259), PIT (i8254), RTC (MC146818).

### Device Tree Topology

QEMU's QOM (QEMU Object Model) device tree is rooted at `main-system-bus`. The tree alternates between **buses** and **devices**: a bus contains devices, each device may expose child buses. For the i440FX machine, the primary path is:

```
main-system-bus
 └─ i440FX (PCI host bridge) ── pci.0 (PCIBus)
      ├─ PIIX3 (PCI-to-ISA bridge) ── isa.0 (ISABus)
      │    ├─ i8259 (PIC master + slave)
      │    ├─ i8254 (PIT)
      │    ├─ MC146818 (RTC)
      │    ├─ Serial (UART 16550A)
      │    └─ PS/2 keyboard + mouse
      ├─ PIIX3-IDE
      ├─ PIIX4-PM (ACPI)
      ├─ VGA (Bochs/stdvga)
      └─ ... (NIC, virtio, etc. from command line)
```

## Machine Initialization

Each QEMU machine type is registered via a `MachineClass` structure whose `init` callback drives board setup. The i440FX PC uses the `DEFINE_I440FX_MACHINE` macro, which ultimately calls **`pc_init1()`**. The initialization sequence is:

1. **`i440fx_init()`** — creates the i440FX PCI host bridge device and returns the root `PCIBus`. Also creates the PIIX3 southbridge on that bus.
2. **`pc_memory_init()`** — maps guest RAM into the flat memory address space and loads the BIOS ROM. The BIOS image is mapped at the top of the 32-bit address space (reset vector region near `0xFFFF0000`) so the CPU can fetch its first instruction there.
3. **Interrupt controllers** — the dual i8259 PIC, the I/O APIC, and per-VCPU Local APICs are created (see [kvm-interrupt-virtualization](kvm-interrupt-virtualization.md)).
4. **Timers** — PIT (i8254), HPET, and the RTC are instantiated.
5. **Legacy I/O** — serial ports (COM1–COM4), PS/2 keyboard and mouse controller.
6. **PCI device instantiation** — devices requested on the QEMU command line (`-device`) are created on the PCI bus.

## The fw_cfg Device

`fw_cfg` is QEMU's mechanism for passing structured data from the hypervisor to the guest firmware without requiring the firmware to probe real hardware.

### Interface

| I/O Port | Width | Purpose |
|----------|-------|---------|
| 0x510 | 16-bit | Selector — write a 16-bit key to choose an item |
| 0x511 | 8-bit | Data — sequential reads return the item's bytes |
| 0x514 | 32-bit | DMA — for bulk transfers (address of DMA descriptor) |

Items are identified by a **16-bit key**. The low 14 bits are the item ID; bit 14 distinguishes architecture-generic vs. architecture-specific items.

### Standard Keys

- `FW_CFG_SIGNATURE` (0x0000) — magic value `"QEMU"`.
- `FW_CFG_RAM_SIZE` (0x0003) — guest RAM size in bytes.
- `FW_CFG_NB_CPUS` (0x0005) — number of VCPUs.
- `FW_CFG_BOOT_DEVICE` — boot order string.

### Named File Interface

`fw_cfg_add_file()` registers an item with a path-like name (e.g., `"etc/acpi/tables"`, `"etc/e820"`). The guest firmware reads a directory listing at `FW_CFG_FILE_DIR` and then selects files by their dynamically assigned keys. SeaBIOS relies heavily on this mechanism to discover ACPI tables, SMBIOS data, and other configuration.

## SeaBIOS Boot Flow

SeaBIOS is the default firmware for the i440FX machine. On CPU reset the processor begins executing at the **reset vector** (`0xFFFFFFF0`, aliased near `0xFFFF0`), which jumps into SeaBIOS code.

### Initialization Sequence

1. **`entry_post()` → `handle_post()`** — early POST: sets up a temporary stack, enables A20, transitions to 32-bit protected mode.
2. **`maininit()`** — the main hardware-discovery phase:
   - PCI bus scan — enumerate and configure BARs.
   - VGA initialization — find and run the VGA option ROM.
   - USB initialization — detect UHCI/OHCI/EHCI/xHCI controllers.
   - Read `fw_cfg` for ACPI tables, SMBIOS, and other QEMU-provided data.
3. **`boot_setup()` → `interactive_bootmenu()` → `do_boot()`** — select a boot device and load the boot sector or EFI image.

### Direct Kernel Boot

With `-kernel`, `-initrd`, and `-append` options, QEMU loads a Linux kernel image directly into guest memory and sets up the boot parameters. SeaBIOS detects this via `fw_cfg` and calls **`load_linux()`**, which jumps into the kernel's setup code, bypassing the normal BIOS disk-boot path. This is valuable for kernel development and testing.

See [boot-process](boot-process.md) for the guest-side Linux boot sequence.

## ACPI Table Generation

QEMU dynamically builds ACPI tables that describe the emulated hardware to the guest OS. The function **`acpi_build()`** constructs an `AcpiBuildTables` structure containing:

| Table | Purpose |
|-------|---------|
| **RSDP** | Root System Description Pointer — entry point for ACPI, contains physical address of RSDT/XSDT |
| **RSDT / XSDT** | Array of pointers to all other ACPI tables (32-bit / 64-bit variants) |
| **FADT** | Fixed ACPI Description Table — PM base I/O port, SCI interrupt number, pointers to DSDT |
| **DSDT** | Differentiated System Description Table — AML bytecode describing hardware topology, power states, PCI routing |
| **MADT** | Multiple APIC Description Table — one entry per Local APIC, one per I/O APIC |
| **HPET** | HPET description table |
| **MCFG** | PCI Express ECAM base address for MMCONFIG access |
| **SRAT / SLIT** | System Resource Affinity Table / Locality Information Table — NUMA topology |

The generated tables are passed to firmware through the `fw_cfg` named file `"etc/acpi/tables"`. The RSDP is placed at `"etc/acpi/rsdp"`. SeaBIOS copies these into guest memory where the guest OS expects to find them.

## Interrupt Controllers

QEMU emulates three layers of interrupt delivery hardware. See [kvm-interrupt-virtualization](kvm-interrupt-virtualization.md) for how KVM accelerates interrupt injection.

### PIC — i8259A (PICCommonState)

The classic dual-8259 configuration:

- **Master PIC**: I/O ports `0x20`–`0x21`, handles IRQ 0–7.
- **Slave PIC**: I/O ports `0xA0`–`0xA1`, handles IRQ 8–15.
- The slave's output is cascaded into the master's **IRQ 2**.

Key registers: **IRR** (Interrupt Request Register), **ISR** (In-Service Register), **IMR** (Interrupt Mask Register). Programming follows the ICW1–ICW4 initialization sequence, then OCW1–OCW3 for runtime control.

### I/O APIC (IOAPICCommonState)

- **24 input pins**, memory-mapped at `0xFEC00000`.
- Each pin has a 64-bit **Redirection Table Entry (RTE)** specifying: interrupt vector, delivery mode (Fixed, Lowest Priority, SMI, NMI, INIT, ExtINT), destination APIC ID, and mask bit.
- Accessed indirectly through an index register (offset 0x00) and a data register (offset 0x10).

### Local APIC (APICCommonState)

- One per VCPU, memory-mapped at `0xFEE00000`.
- **Local Vector Table (LVT)**: configures local interrupt sources (timer, thermal, performance counters, LINT0/LINT1, error).
- **ICR** (Interrupt Command Register): for sending IPIs (inter-processor interrupts).
- **APIC timer**: see Timers section below.
- **EOI register**: guest writes to signal end-of-interrupt.

## Timers

### PIT — i8254 (PITCommonState)

- Three independent 16-bit counters at I/O ports `0x40`–`0x43`.
- **Channel 0**: system timer, wired to IRQ 0 (PIC) or I/O APIC pin 2. Base frequency: **1,193,182 Hz** (derived from the original PC's 14.31818 MHz / 12 crystal).
- Channel 1: historically DRAM refresh (unused in QEMU).
- Channel 2: PC speaker tone generation.
- Supports modes 0–5 (interrupt on terminal count, one-shot, rate generator, square wave, etc.).

### HPET (HPETState)

- Memory-mapped at `0xFED00000`.
- Up to **8 independent timers**, each with its own comparator and interrupt routing.
- A single 64-bit **main counter** incremented at a fixed frequency (minimum 10 MHz per spec).
- Each timer supports **periodic** and **one-shot** modes.
- Intended as a replacement for the PIT with higher precision and more channels.

### Local APIC Timer

- Built into each Local APIC, thus per-VCPU.
- Three modes: **one-shot** (count down once), **periodic** (auto-reload), **TSC-deadline** (fire when TSC reaches a programmed value — used by modern kernels for tickless operation).

## VGA Emulation

QEMU emulates a VGA-compatible display adapter (based on the **Bochs VBE / stdvga** design) via `VGACommonState`.

### Register Model

Standard VGA register sets accessed through I/O ports:

- **SR** (Sequencer, 0x3C4–0x3C5) — clocking, memory mode, character map select.
- **CR** (CRT Controller, 0x3D4–0x3D5) — horizontal/vertical timing, cursor position, start address.
- **GR** (Graphics Controller, 0x3CE–0x3CF) — read/write mode, bit mask, color compare.
- **AR** (Attribute Controller, 0x3C0–0x3C1) — palette, overscan, pixel panning.
- **DAC** (0x3C6–0x3C9) — 256-entry color palette (6 bits per channel).

Register reads and writes are handled by `vga_ioport_read()` and `vga_ioport_write()`.

### Memory Access and Planar Model

The VGA framebuffer at `0xA0000`–`0xBFFFF` uses the classic **planar memory model** (4 planes, controlled by map mask and read map select). QEMU emulates this through `vga_mem_readb()` and `vga_mem_writeb()`, which apply the plane logic, bit masks, and ALU operations specified by the graphics controller registers.

### VBE (Bochs) Extensions

The Bochs VBE interface (I/O ports `0x1CE`–`0x1CF`) provides high-resolution, linear-framebuffer modes beyond standard VGA. The guest driver sets resolution, color depth, and enables the linear framebuffer via VBE registers, bypassing the planar model entirely.

### Display Pipeline

```
vram_ptr (guest framebuffer)
  → vga_update_display()
    → vga_draw_graphic()  [dirty page tracking skips unchanged regions]
      → DisplaySurface (host-side pixel buffer)
        → QemuConsole
          → DisplayChangeListener (SDL / GTK / VNC / Spice backend)
```

`vga_update_display()` is called periodically by QEMU's GUI refresh timer. Dirty page tracking ensures only modified regions of VRAM are re-rendered.

## Serial Port — UART 16550A

QEMU emulates the industry-standard **16550A UART** (`SerialState`), with COM1 at I/O base `0x3F8`, IRQ 4.

### Register Map (offsets from base)

| Offset | Read | Write |
|--------|------|-------|
| 0 | RBR (Receive Buffer) | THR (Transmit Holding) |
| 1 | IER (Interrupt Enable) | IER |
| 2 | IIR (Interrupt ID) | FCR (FIFO Control) |
| 3 | LCR (Line Control) | LCR |
| 4 | MCR (Modem Control) | MCR |
| 5 | LSR (Line Status) | — |
| 6 | MSR (Modem Status) | — |
| 7 | SCR (Scratch) | SCR |

The 16550A adds a **16-byte FIFO** (enabled via FCR) to reduce interrupt overhead compared to the original 8250.

### Backend

The emulated UART connects to a **Chardev** backend in QEMU, which can be:

- `stdio` — QEMU's own terminal (default for `-nographic`).
- `pty` — allocates a host pseudo-terminal.
- `socket` — TCP/Unix socket for remote console access.
- `file` — log output to a file.

The serial console is the primary interface for kernel debugging (`console=ttyS0`) and is essential when running QEMU in headless mode.

## See also

- [qemu-kvm-overview](qemu-kvm-overview.md)
- [kvm-cpu-virtualization](kvm-cpu-virtualization.md)
- [kvm-interrupt-virtualization](kvm-interrupt-virtualization.md)
- [boot-process](boot-process.md)
- [device-driver-model](device-driver-model.md)
