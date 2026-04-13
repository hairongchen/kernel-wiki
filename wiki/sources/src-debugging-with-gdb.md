---
type: source
created: 2026-04-09
updated: 2026-04-09
sources: [debugging-with-gdb]
tags: [gdb, debugger, debugging, gnu, development-tools, source-summary]
---

# Debugging with GDB

**Authors:** Richard Stallman, Roland Pesch, Stan Shebs, et al.
**Publisher:** Free Software Foundation, 2013
**Covers:** GDB version 7.6.50.20130313 (Tenth Edition)
**ISBN:** 978-0-9831592-3-0

## Overview

The official GNU GDB manual and the definitive reference for the GNU Debugger. Covers every feature from basic breakpoints through reverse debugging, remote target protocols, tracepoints, and the Python scripting API. While [gnu-toolchain](../entities/gnu-toolchain.md) (from "Advanced Linux Programming") introduces GDB's most common commands, this source provides exhaustive coverage of the entire debugger. The remote debugging and tracepoint chapters are particularly relevant to kernel development — GDB connects to QEMU's built-in gdbserver for VM debugging, and the tracepoint mechanism relates to the kernel's ftrace/kprobes infrastructure.

## Chapter Summaries

### Ch 1: A Sample GDB Session

A quick walkthrough demonstrating the core debugging loop: setting breakpoints, running a program, inspecting variables, stepping through code, and examining the call stack. Establishes the interactive command-line model that the rest of the manual builds upon.

### Ch 2: Getting In and Out of GDB

Invocation options and startup sequence:

- **Key flags** — `-batch` (non-interactive scripting), `-tui` (Text User Interface), `-interpreter=mi` (machine interface for IDEs), `-q` (suppress banner), `-x file` (execute commands from file), `--args` (pass arguments to inferior), `-readnow` (load all symbols eagerly).
- **Startup sequence** — GDB reads system `gdbinit`, then user `~/.gdbinit`, then directory `.gdbinit` (subject to auto-loading safe-path).
- **Quitting** — `quit` or `Ctrl-D`. GDB prompts if the inferior is still running.
- **Shell commands** — `shell cmd` executes a shell command without leaving GDB.
- **Logging** — `set logging on` redirects output to `gdb.txt` for session recording.

### Ch 3: GDB Commands

Command syntax, abbreviation rules, and the help system. Tab completion works for commands, function names, variable names, and filenames. The `help` command is organized by category (breakpoints, data, files, running, etc.). Commands can be abbreviated to unique prefixes.

### Ch 4: Running Programs Under GDB

- **Compilation** — requires `-g` (debug info); `-g3` adds macro definitions; `-O0` recommended for faithful debugging.
- **Starting** — `run`, `start` (break at main). Attaching to running processes with `attach pid`, detaching with `detach`.
- **Arguments and environment** — `set args`, `show args`, `set environment`, `unset environment`, I/O redirection.
- **Multi-inferior** — `add-inferior` and `clone-inferior` for debugging multiple programs simultaneously. Each inferior has independent address space.
- **Multi-threaded debugging** — `info threads`, `thread N`, `thread apply all bt` for applying commands across threads.
- **Fork debugging** — `set follow-fork-mode child/parent`, `set detach-on-fork off` to debug both parent and child.
- **Checkpoints** — `checkpoint` saves a snapshot of program state; `restart N` restores it. Implemented via fork.

### Ch 5: Stopping and Continuing

The most feature-rich chapter — covers all mechanisms for controlling program execution:

- **Breakpoints** — `break` (location), `tbreak` (temporary), `hbreak` (hardware), `rbreak` (regex on function names), pending breakpoints for unloaded shared libraries.
- **Watchpoints** — `watch expr` (write), `rwatch expr` (read), `awatch expr` (access). Hardware watchpoints are fast but limited in number; software watchpoints are slow but unlimited.
- **Catchpoints** — `catch syscall`, `catch fork`, `catch exec`, `catch throw`, `catch load` (shared library), `catch signal`.
- **Management** — `delete`, `disable`, `enable`, conditions (`condition N expr`), ignore counts (`ignore N count`), command lists (`commands N ... end`), `save breakpoints file`.
- **Dynamic printf** — `dprintf location, format, args` inserts printf-like output without modifying the program. Styles: `gdb` (GDB-side), `call` (calls printf in inferior), `agent` (bytecode on target).
- **Stepping** — `step` (into), `next` (over), `finish` (out of frame), `until` (to line/address), `advance` (to location), `stepi`/`nexti` (single instruction).
- **Skipping** — `skip function`/`skip file` to avoid stepping into uninteresting code (e.g., library internals).
- **Signal handling** — `handle SIGNAME stop/nostop print/noprint pass/nopass` controls how GDB intercepts signals.
- **Execution modes** — all-stop mode (all threads stop when one hits a breakpoint) vs. non-stop mode (only the hitting thread stops). `set scheduler-locking on/step/off` controls which threads run during stepping. Background execution with `continue&`, `step&`, etc.
- **Thread-specific breakpoints** — `break location thread N` restricts a breakpoint to a specific thread.
- **Observer mode** — read-only debugging that avoids modifying the target (no breakpoint insertion, no signal injection).

### Ch 6-7: Reverse Execution and Process Record

- **Reverse execution** — `reverse-continue`, `reverse-step`, `reverse-next`, `reverse-finish`. `set exec-direction reverse` makes all execution commands run backward.
- **Record full** — records every instruction and memory change for deterministic replay. High overhead but complete.
- **Record btrace** — uses Intel Processor Trace hardware for low-overhead recording. Supports `instruction-history` and `function-call-history` commands.
- **Save/restore** — `record save`/`record restore` for persisting execution logs.

See [concept-reverse-debugging](../concepts/concept-reverse-debugging.md).

### Ch 8: Examining the Stack

- **Backtrace** — `bt` (summary), `bt full` (with local variables), `bt N` (innermost N frames), `bt -N` (outermost N frames).
- **Frame selection** — `frame N`, `up`, `down` to navigate the call stack.
- **Frame inspection** — `info frame` (detailed frame metadata: saved registers, return address), `info args`, `info locals`.

### Ch 9: Examining Source Files

- **Listing** — `list` (source lines), `list function`, `list file:line`.
- **Location specs** — `file:line`, `function`, `*address` (code at arbitrary address).
- **Source path** — `directory dir` adds search directories; `set substitute-path from to` remaps source paths for moved builds.
- **Disassembly** — `disassemble` with `/m` (interleaved source), `/r` (raw instruction bytes). `set disassembly-flavor att/intel` selects syntax style.

### Ch 10: Examining Data

The primary data inspection chapter:

- **Print** — `print expr` with format codes: `x` (hex), `d` (decimal), `u` (unsigned), `o` (octal), `t` (binary), `a` (address), `c` (char), `f` (float), `s` (string), `r` (raw, bypass pretty-printer).
- **Memory examination** — `x/nfu addr`: `n` repeat count, `f` format, `u` unit size (b/h/w/g for 1/2/4/8 bytes).
- **Automatic display** — `display expr` re-evaluates and prints at every stop.
- **Print settings** — 20+ options: `set print pretty`, `set print array`, `set print elements N`, `set print object` (RTTI-based type), `set print vtbl`, `set print demangle`, etc.
- **Value history** — `$1`, `$2`, ... for previous print results. `$` is the last value, `$$` the one before.
- **Convenience variables** — `$_` (last address examined by `x`), `$__` (last value from `x`), `$_siginfo` (signal details), user-defined `$var`.
- **Registers** — `$pc`, `$sp`, `$fp`, `info registers`, `info all-registers`.
- **Pretty-printing** — framework for type-aware display (e.g., showing STL containers as readable structures instead of raw pointers).
- **Memory search** — `find start, end, value` searches memory for byte patterns.

### Ch 11: Optimized Code

Debugging challenges with compiler-optimized code: inline function handling (GDB can set breakpoints on inlined instances), tail call frame reconstruction when `-O2` eliminates frame pointers.

### Ch 12: C Preprocessor Macros

`macro expand EXPR` and `info macro NAME` inspect preprocessor macro definitions and expansions. Requires compiling with `-g3` to embed macro information in debug info.

### Ch 13: Tracepoints

Non-intrusive data collection for production-like environments:

- **Setting** — `trace` (normal), `ftrace` (fast, uses in-process agent), `strace` (static, uses markers compiled into the program).
- **Actions** — `collect expr` (record data), `teval expr` (evaluate without collecting), `while-stepping N` (collect at each step for N steps).
- **Trace state variables** — `tvariable $var` for persistent counters/accumulators across tracepoint hits.
- **Control** — `passcount N` (auto-stop after N hits), `tstart`/`tstop`/`tstatus`.
- **Analysis** — `tfind` navigates collected frames, `tdump` displays frame data, `tsave` exports to file.
- **Modes** — circular buffer mode, disconnected tracing (continues after GDB disconnects).

### Ch 14: Debugging Programs That Use Overlays

Support for overlay debugging in memory-constrained embedded systems where code sections are swapped in and out of a single memory region. GDB tracks which overlay is currently mapped.

### Ch 15: Using GDB with Different Languages

Language-specific support:

- **C/C++** — name demangling, vtable display, RTTI-based type identification, operator overloading in expressions.
- **Other languages** — D, Go, Objective-C, Fortran, Pascal, Modula-2, Ada (with special support for tasks, exceptions, and Ada attributes).

### Ch 16: Examining the Symbol Table

- **Queries** — `info address sym`, `info symbol addr`, `whatis expr` (brief type), `ptype expr` (full type with members; `/o` flag shows struct layout offsets and holes), `info functions`, `info variables`, `info types`, `info scope`.
- **Macro inspection** — `macro expand` for preprocessor expansion.

### Ch 17: Altering Execution

- **Modify state** — `set variable VAR = VALUE`, `set {int}0xADDR = VALUE`.
- **Control flow** — `jump location` (resume at arbitrary point), `signal SIG` (deliver signal), `return expr` (force return from current frame).
- **Calling functions** — `call func(args)` invokes a function in the inferior.
- **Compile and inject** — `compile code` uses GCC to compile C code at runtime and inject it into the running program.

### Ch 18: GDB Files

- **Loading** — `file`, `exec-file`, `symbol-file`, `add-symbol-file` for separate loading of executable and symbols.
- **Separate debug info** — `.gnu_debuglink` section with CRC32 checksum, or build-ID-based lookup via `.note.gnu.build-id`. Standard search paths: `/usr/lib/debug/`.
- **Index acceleration** — `.gdb_index` section provides a pre-built symbol hash table for fast loading of large binaries.

### Ch 19-20: Specifying a Debugging Target / Remote Debugging

- **Targets** — `target remote host:port` (exclusive connection), `target extended-remote` (persistent connection supporting `run`).
- **gdbserver** — `gdbserver host:port program`, `--attach pid`, `--multi` (multi-process), `--once` (exit after disconnect). Supports TCP and serial connections.
- **Remote stubs** — for bare-metal targets: implement `set_debug_traps()`, `handle_exception()`, `breakpoint()`.
- **File transfer** — `remote put/get/delete` for transferring files to the target.
- **monitor command** — `monitor cmd` sends pass-through commands to the remote stub.

See [gdb-remote-debugging](../entities/gdb-remote-debugging.md).

### Ch 21: Configuration-Specific Information

Platform-specific features:

- **x86** — `set disassembly-flavor`, process record support, SSE/AVX register display, MPX bounds checking.
- **ARM, MIPS, PowerPC** — architecture-specific register sets and calling conventions.
- **SVR4/Linux** — `info proc mappings` (memory map), `info proc status` (process status from `/proc`).

### Ch 22: Controlling GDB

- **UI** — custom prompt, command editing (readline), history, pagination.
- **Number formatting** — `set input-radix`/`set output-radix` for hex/octal/decimal defaults.
- **Auto-loading** — `set auto-load safe-path` controls which directories GDB will auto-load scripts from (security measure against malicious `.gdbinit` files).
- **Debug output** — `set debug SUBSYSTEM on` for GDB internal diagnostics.

### Ch 23: Extending GDB

The extensibility chapter — one of the most important for power users:

- **User commands** — `define cmd ... end`, `document cmd ... end`. Control flow: `if/else/end`, `while/end`.
- **Hooks** — `hook-stop` runs commands at every stop, `hook-CMD`/`hookpost-CMD` wrap existing commands.
- **Command files** — `source file` executes a GDB script. `.gdbinit` loaded at startup.
- **Python API** (extensive):
  - `gdb.Value` — wraps inferior values with type-aware operations.
  - `gdb.Type` — type introspection with `TYPE_CODE_*` constants (STRUCT, UNION, PTR, ARRAY, etc.).
  - `gdb.Breakpoint` — scripted breakpoints with custom `stop()` method returning True/False.
  - `gdb.Frame` — stack frame access.
  - `gdb.Inferior` — process-level operations (read/write memory, threads).
  - `gdb.Command` — define new GDB commands in Python with completion support.
  - `gdb.Parameter` — define new `set`/`show` parameters.
  - **Pretty-printing framework** — register type-specific printers for structured display.
  - **Frame filters/decorators** — transform backtrace output programmatically.
  - **Frame unwinders** — teach GDB to unwind custom stack frame layouts.
  - **Xmethods** — Python-implemented "methods" that override inferior function calls (e.g., for optimized-out methods).
  - **Events API** — subscribe to stop, continue, exit, new-objfile, and other events.
- **Auto-loading security** — safe-path restricts which Python scripts GDB will auto-load.

### Ch 24-26: Command Interpreters, TUI, and Emacs

- **Interpreters** — `console` (interactive), `mi`/`mi2` (machine interface for IDE integration).
- **TUI** — Text User Interface with `src`, `asm`, `cmd`, `regs` windows. `layout src/asm/split/regs`. SingleKey mode maps single keystrokes to common commands.
- **Emacs** — `M-x gdb` launches GDB within Emacs with GUD (Grand Unified Debugger) keybindings for integrated source-level debugging.

### Ch 27: The GDB/MI Interface

Machine-readable protocol for IDE frontends:

- **Record types** — result records (`^done`, `^error`), async records (`*stopped`, `=thread-created`), stream records (`~` console, `@` target, `&` log).
- **Command categories** — breakpoint, thread, stack, variable-object, data-manipulation, tracepoint, file, and target commands.
- **Variable objects** — stateful handles for efficient IDE variable tracking without re-evaluating expressions at every stop.

### Ch 28-31: Annotations, JIT, In-Process Agent, Reporting Bugs

- **Annotations** — legacy front-end protocol (superseded by GDB/MI).
- **JIT interface** — programs register JIT-compiled code via `jit_code_entry`/`jit_descriptor` structures and call `__jit_debug_register_code()` to notify GDB.
- **In-process agent** — `libinproctrace.so` enables fast tracepoints using jump pads injected into the inferior's address space.
- **Bug reporting** — how to file useful GDB bug reports.

### Ch 32-33: Using History Interactively / Readline

Readline library integration: Emacs and vi editing modes, key bindings, `~/.inputrc` configuration, kill ring for cut/paste, history expansion with `!` (event designators, word designators, modifiers).

### Appendices

- **A: Installing GDB** — `configure`/`make`/`make install` build process.
- **B: Maintenance Commands** — `maint agent` (compile agent expressions), `maint info breakpoints/sections`, `maint print architecture/registers`, `maint set dwarf2`, `maint demangle`.
- **C: Remote Serial Protocol (RSP)** — packet format `$data#checksum`, `+`/`-` acknowledgment, run-length encoding. Key packets: `g`/`G` (read/write all registers), `p`/`P` (single register), `m`/`M`/`X` (memory), `c`/`s` (continue/step), `Z`/`z` (breakpoints/watchpoints), `vCont` (per-thread control), `qSupported` (capability negotiation). Stop replies: `T`/`S`/`W`/`X`. Non-stop mode uses async notifications.
- **D: Agent Expressions** — stack-based bytecode VM for evaluating tracepoint expressions on the target: `trace`, `trace_quick`, `reg`, `getv`/`setv` opcodes.
- **E: Target Descriptions** — XML format for register discovery, with standard features defined per architecture.
- **F: .gdb_index Format** — CU list + address area + symbol hash table + constant pool for fast symbol lookup.
- **G: Trace File Format** — TFILE format with R (register), M (memory), V (variable) blocks.
- **H-I: Licenses** — GPL and FDL texts.

## Key Themes

1. **Comprehensive debugging primitives** — breakpoints, hardware watchpoints, catchpoints, tracepoints, and dynamic printf each target different debugging scenarios, from interactive exploration to production monitoring.
2. **Multi-level execution control** — all-stop vs non-stop mode, scheduler-locking, background execution, and observer mode provide fine-grained control over multi-threaded debugging behavior.
3. **Reverse debugging** — process record and replay enables backward execution, fundamentally changing how intermittent and timing-dependent bugs can be diagnosed.
4. **Remote debugging architecture** — the Remote Serial Protocol (RSP) enables debugging across any communication channel, critical for embedded systems and kernel debugging via KGDB.
5. **Extensibility** — the Python scripting API provides programmatic access to nearly all GDB internals, enabling custom commands, pretty-printers, frame unwinders, and automated debugging workflows (e.g., the kernel's `scripts/gdb/` `lx-dmesg`/`lx-ps` commands).
6. **Machine interface** — GDB/MI provides a structured protocol for IDE integration, with variable objects for efficient state tracking.

## Relationship to Other Sources

This manual is the definitive GDB reference. The [gnu-toolchain](../entities/gnu-toolchain.md) page (from "Advanced Linux Programming") provides a practical overview of GDB's most common commands; this source covers every feature exhaustively. The remote debugging chapters are directly relevant to the QEMU/KVM virtualization coverage in [src-qemu-kvm-source-code-and-application](src-qemu-kvm-source-code-and-application.md) and [src-mastering-kvm-virtualization](src-mastering-kvm-virtualization.md) — GDB connects to QEMU's built-in gdbserver for VM debugging. The tracepoint mechanism relates to the kernel's ftrace/kprobes infrastructure. The Python API enables kernel-aware debugging scripts in the kernel source tree's `scripts/gdb/` directory.

## See also

- [gdb-debugger](../entities/gdb-debugger.md)
- [gdb-remote-debugging](../entities/gdb-remote-debugging.md)
- [gnu-toolchain](../entities/gnu-toolchain.md)
- [concept-reverse-debugging](../concepts/concept-reverse-debugging.md)
