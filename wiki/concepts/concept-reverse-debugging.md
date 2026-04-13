---
type: concept
created: 2026-04-09
updated: 2026-04-09
sources: [debugging-with-gdb]
tags: [debugging, reverse-debugging, record-replay, gdb, process-record]
---

# Reverse Debugging and Execution Recording

## Overview

Reverse debugging allows a developer to execute a program backward — undoing instructions, restoring register and memory state — to find the root cause of a bug by working backward from the symptom. Instead of repeatedly restarting and adding breakpoints to narrow down a bug, the developer runs forward past the problem, then steps backward to find exactly where state went wrong.

## The Problem

Traditional debugging is forward-only: when you step past the critical moment, you must restart and try again. For bugs that are intermittent, timing-dependent, or require complex setup to reproduce, this can be extremely time-consuming. Reverse debugging eliminates this by letting you rewind execution.

## Recording Mechanisms

### Software Record (GDB `record full`)

- GDB records every instruction's effects: register changes, memory writes
- `record full` — start recording; GDB logs all state changes
- Execution slows significantly (hundreds of times) because GDB must single-step and log every instruction
- `set record full insn-number-max N` — limit log size (default 200000 instructions); 0 = unlimited
- `set record full stop-at-limit on|off` — behavior when limit reached
- After recording, the log can be saved/restored: `record save filename`, `record restore filename`
- The log enables both forward replay and true reverse execution

### Hardware-Assisted Recording (GDB `record btrace`)

- Uses Intel Processor Trace (Intel PT) or Branch Trace Store (BTS)
- Hardware traces branches with minimal performance overhead
- Cannot record memory changes — no reverse execution support, only instruction/function history
- `record btrace` — start hardware-assisted recording
- `record instruction-history` — list recorded instructions
- `record function-call-history [/l] [/i]` — list function calls; /l shows source lines, /i shows instruction ranges

## Reverse Execution Commands

All require an active `record full` session:

| Command | Abbreviation | Behavior |
|---------|-------------|----------|
| `reverse-continue` | `rc` | Run backward until a breakpoint, watchpoint, or the beginning of the recording |
| `reverse-step [N]` | | Step backward into function calls (reverse of `step`) |
| `reverse-next [N]` | | Step backward over function calls (reverse of `next`) |
| `reverse-stepi [N]` | | Reverse one machine instruction |
| `reverse-nexti [N]` | | Reverse one instruction, stepping over calls |
| `reverse-finish` | | Run backward until the current function was called |

### Execution Direction

- `set exec-direction reverse` — all execution commands (step, next, continue, finish) operate backward
- `set exec-direction forward` — restore normal operation
- `return` command cannot be used in reverse mode

## Record and Replay Model

GDB's process record operates in two modes:

**Record mode**: The inferior executes normally on real hardware; GDB intercepts and logs every instruction's side effects (register and memory changes). The program runs at reduced speed.

**Replay mode**: When the user moves backward (or uses `record goto`), the inferior does NOT actually execute. Instead, GDB replays state changes from the log. Registers and memory are restored to their recorded values. From the program's perspective, time moves backward.

Key commands:

- `record goto begin|end|N` — jump to a point in the recording
- `record delete` — while in the past, delete the future log and start recording new execution from the current point (creates an alternate timeline)
- `info record` — statistics: mode (record/replay), instruction count, log limits

## Checkpoints (Alternative Approach)

GDB also supports checkpoints on GNU/Linux — full process snapshots:

- `checkpoint` — fork the inferior to create a snapshot
- `restart checkpoint-id` — switch to a saved snapshot (restores all memory, registers, file positions)
- `info checkpoints` / `delete checkpoint id`
- Lighter-weight than full recording; useful for "save state before risky operation"
- Limitation: cannot undo I/O to external devices, file writes are not reversed

## Watchpoints + Reverse Execution

The most powerful pattern: combine a watchpoint with reverse-continue to find when a variable was corrupted:

1. Observe the corrupted value
2. `watch variable` — set a watchpoint
3. `reverse-continue` — run backward until the watchpoint triggers
4. GDB stops at the exact instruction that last modified the variable

This directly answers "who changed this value?" without guessing or adding printf statements.

## Tracepoints (Non-Intrusive Alternative)

When stopping is unacceptable (real-time systems, timing-sensitive bugs), tracepoints collect data without halting:

- `trace`/`ftrace`/`strace` — set collection points
- `collect` actions gather registers, memory, expressions
- Trace state variables provide cross-hit state (counters, accumulators)
- Data examined offline with `tfind`/`tdump`
- Can operate disconnected from GDB (`set disconnected-tracing on`)

## Performance and Limitations

| Method | Speed Impact | Reverse Exec | Memory Cost | Platform |
|--------|-------------|--------------|-------------|----------|
| `record full` | ~100-1000x slower | Full support | High (log grows) | Any |
| `record btrace` | Minimal (~5%) | No (history only) | Low (HW buffer) | Intel PT/BTS |
| Checkpoints | None during run | Snapshot restore | Per-checkpoint fork | GNU/Linux |
| Tracepoints | Minimal (fast) | No | Trace buffer | gdbserver/IPA |

Limitations of `record full`:

- Significant performance overhead (every instruction is single-stepped)
- Log size grows with execution length
- System calls that modify external state cannot be truly reversed
- Not supported for all targets/architectures

## Relationship to Kernel Debugging

- KGDB provides a GDB stub in the Linux kernel for remote debugging
- QEMU's built-in gdbserver can debug guest kernels
- Process record is most useful for user-space applications
- For kernel debugging, tracepoints (ftrace, kprobes) and hardware trace (Intel PT via `perf`) serve similar investigative goals

## See also

- [gdb-debugger](../entities/gdb-debugger.md)
- [gdb-remote-debugging](../entities/gdb-remote-debugging.md)
- [gnu-toolchain](../entities/gnu-toolchain.md)
