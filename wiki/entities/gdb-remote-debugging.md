---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [debugging-with-gdb]
tags: [gdb, remote-debugging, gdbserver, rsp, serial-protocol, kernel-debugging]
---

# GDB Remote Debugging

## Overview

GDB's remote debugging architecture separates the debugger (GDB on the host) from the target being debugged, communicating over the GDB Remote Serial Protocol (RSP). This enables debugging across serial lines, TCP/IP, and pipes -- critical for embedded systems, kernel debugging via KGDB, and VM debugging via QEMU's built-in gdbserver.

## gdbserver

### Starting gdbserver

- `gdbserver host:port program [args]` -- start program under gdbserver
- `gdbserver --attach host:port pid` -- attach to running process
- `gdbserver --multi host:port` -- multi-process mode (no initial program; GDB can run/attach later)
- `gdbserver --once host:port program` -- exit after first connection disconnects
- `gdbserver --wrapper wrapper -- program` -- run under a wrapper (e.g., valgrind)
- Serial: `gdbserver /dev/ttyS0 program`

### Connecting from GDB

- `target remote host:port` -- connect to gdbserver (begins debugging immediately)
- `target extended-remote host:port` -- extended mode (supports `run`, `attach`, `detach`; persists across program exits)
- `target remote | command` -- pipe transport (e.g., `target remote | ssh host gdbserver - program`)
- `detach` -- disconnect, let remote continue
- `disconnect` -- disconnect without affecting target state
- `monitor command` -- pass command to gdbserver for local execution

### Multi-Process Mode

With `--multi`, gdbserver persists and accepts multiple run/attach cycles. GDB sends `vRun;filename;arg1;arg2` to start programs, `vAttach;pid` to attach, and `vKill;pid` to kill specific processes.

## Remote Serial Protocol (RSP)

### Packet Format

- Every packet: `$packet-data#XX` where XX is a 2-hex-digit checksum (sum of bytes mod 256)
- Acknowledgment: `+` (OK) / `-` (retransmit)
- `QStartNoAckMode` -- disable ack for performance over reliable transports (TCP)
- Run-length encoding: `*` followed by a character encodes repetition (character ASCII - 29 = count)
- Binary data escaping: `}` XORs next byte with 0x20; must escape `}`, `#`, `$`, `*`

### Core Packets

| Packet | Purpose |
|--------|---------|
| `g` / `G XX...` | Read / write all general registers |
| `p n` / `P n=val` | Read / write register n |
| `m addr,len` / `M addr,len:XX` | Read / write memory (hex) |
| `X addr,len:data` | Write memory (binary, with escaping) |
| `c [addr]` / `s [addr]` | Continue / single-step [from addr] |
| `C sig[;addr]` / `S sig[;addr]` | Continue / step with signal |
| `?` | Report halt reason |
| `k` | Kill target |
| `D [;pid]` | Detach [from process] |
| `H op thread-id` | Set thread for operation (c=continue, g=general) |
| `T thread-id` | Check if thread alive |
| `Z type,addr,kind` / `z type,addr,kind` | Insert / remove breakpoint/watchpoint |

### Breakpoint/Watchpoint Packets

- Z0/z0: software breakpoint, Z1/z1: hardware breakpoint
- Z2/z2: write watchpoint, Z3/z3: read watchpoint, Z4/z4: access watchpoint
- `kind` is architecture-specific (e.g., instruction length for ARM Thumb vs ARM)
- Conditional breakpoints: `Z0,addr,kind;Xcond-len,cond-bytecode` (agent expression evaluated on stub)

### Stop Replies

- `S AA` -- stopped by signal AA
- `T AA n1:r1;n2:r2;...` -- stopped with register/status info. Key fields:
  - `thread:id` -- which thread stopped
  - `core:N` -- CPU core number
  - `watch:addr` / `rwatch:addr` / `awatch:addr` -- watchpoint trigger
  - `library:` -- shared library event
  - `fork:pid` / `vfork:pid` -- fork events
- `W AA` -- process exited with status
- `X AA` -- process terminated by signal
- `O hex-data` -- console output from inferior
- `F call-id,params` -- file-I/O syscall request

### Advanced Execution Control (vCont)

`vCont;action[:thread-id]...` provides per-thread resume control. Actions: `c` (continue), `s` (step), `C sig` (continue+signal), `S sig` (step+signal), `t` (stop thread), `r start,end` (range step).

### Capability Negotiation (qSupported)

GDB and stub exchange feature lists. Key features:

- `PacketSize=N` -- max packet size
- `multiprocess+` -- thread IDs become p pid.tid
- `QNonStop+` -- non-stop mode support
- `ConditionalBreakpoints+` -- stub evaluates conditions
- `qXfer:features:read+` -- target description XML
- `qXfer:memory-map:read+` -- memory map

### Thread Enumeration

- `qfThreadInfo` / `qsThreadInfo` -- enumerate threads (first batch / subsequent)
- `qThreadExtraInfo,id` -- human-readable thread description
- `qC` -- current thread ID

### Non-Stop Mode

- `QNonStop:1` -- enable (threads run independently)
- Stop events delivered asynchronously as `%Stop:T05...` notifications
- `vStopped` -- GDB acknowledges each notification; stub sends next pending or OK

### File-I/O Extension

The target stub proxies file operations to the GDB host via F request packets. `Fsyscall-name,params` causes GDB to execute the syscall on the host, returning `Fretcode,errno,Ctrl-C`. Supported syscalls: open, close, read, write, lseek, rename, unlink, stat, fstat, gettimeofday, isatty, system. All data types use big-endian encoding regardless of target byte order.

### Host I/O Packets

`vFile:open/close/pread/pwrite/unlink/readlink/setfs` -- GDB accesses the remote filesystem. Used for `remote get/put/delete` file transfer commands.

## Remote Stubs (Bare-Metal Targets)

For targets without an OS, a remote stub is linked into the target program:

1. `set_debug_traps()` -- initialize stub, install trap handlers
2. `handle_exception(int signo)` -- called on trap; communicates with GDB (read/write registers/memory, continue/step)
3. `breakpoint()` -- programmatic breakpoint entry

Minimum packets the stub must handle: `g` (read regs), `G` (write regs), `m` (read mem), `M` (write mem), `c` (continue), `s` (step).

## Agent Expressions

Stack-based bytecode VM for evaluating expressions on the target (tracepoint conditions/actions):

- 64-bit stack entries, sign-extended constants, zero-extended memory loads
- Arithmetic: add, sub, mul, div_signed/unsigned, neg
- Bitwise: bit_and/or/xor/not, lsh, rsh_signed/unsigned
- Memory: ref8/16/32/64 (read from target address)
- Control: goto, if_goto, end
- Tracepoint: trace (record memory), reg (read register), getv/setv/tracev (trace state variables)

## Target Descriptions (XML)

Targets report register layouts via `qXfer:features:read:target.xml`:

```xml
<target version="1.0">
  <architecture>i386:x86-64</architecture>
  <feature name="org.gnu.gdb.i386.core">
    <reg name="rax" bitsize="64" regnum="0" type="int64"/>
    ...
  </feature>
</target>
```

Standard features are defined for: i386, AArch64, ARM, MIPS, PowerPC, M68K, and TMS320C6x.

## .gdb_index Section

Accelerated symbol lookup embedded in ELF files:

- Layout: header + CU list + types CU list + address area + symbol hash table + constant pool
- DJB hash with linear probing; entries encode CU index + symbol kind (type/variable/function) + global/static
- Created with `save gdb-index dir` + `objcopy --add-section .gdb_index=file`
- Dramatically speeds up startup for large binaries

## Configuration

- `set remotetimeout N` -- timeout in seconds
- `set remotelogfile file` -- log all RSP traffic (invaluable for protocol debugging)
- `set serial baud N` -- baud rate for serial connections
- `set remote PACKET-packet on|off|auto` -- enable/disable specific protocol packets
- `set tcp auto-retry on`, `set tcp connect-timeout N`

## See also

- [gdb-debugger](gdb-debugger.md)
- [gnu-toolchain](gnu-toolchain.md)
- [qemu-kvm-overview](qemu-kvm-overview.md)
- [kvm-cpu-virtualization](kvm-cpu-virtualization.md)
- [concept-reverse-debugging](../concepts/concept-reverse-debugging.md)
