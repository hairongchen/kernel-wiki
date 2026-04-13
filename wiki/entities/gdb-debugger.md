---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [debugging-with-gdb, advanced-linux-programming]
tags: [gdb, debugger, debugging, breakpoints, watchpoints, stepping, gnu]
---

# GDB (GNU Debugger)

## Overview

GDB (GNU Debugger) is the standard debugger for GNU/Linux systems, supporting source-level debugging for C, C++, and many other languages. Version 7.6 (covered here) supports breakpoints, watchpoints, catchpoints, tracepoints, reverse execution, multi-threaded/multi-process debugging, Python scripting, and remote debugging via the GDB Remote Serial Protocol. GDB is part of the [gnu-toolchain](gnu-toolchain.md) and is the primary tool for diagnosing bugs in user-space programs and, via KGDB, the Linux kernel itself.

## Invocation

- `gdb program` -- debug an executable
- `gdb program core` -- examine a core dump
- `gdb -p pid` -- attach to a running process
- `gdb --args program arg1 arg2` -- pass arguments to the inferior

Key options: `-batch` (non-interactive), `-tui` (curses UI), `-q` (quiet), `-x file` (execute commands from file), `-ex cmd` (execute a single command), `-readnow` (load all symbols upfront), `--interpreter=mi` (machine interface for IDE integration).

Startup sequence: system gdbinit, then `~/.gdbinit`, then command-line options, then `./.gdbinit` in the working directory, then auto-load scripts, then `-x`/`-ex` commands.

Compiling for debug: `gcc -g` produces DWARF debug info. `-g3` adds macro definitions. `-ggdb` enables GDB-specific extensions. Optimization flags (`-O2`) may cause variables to be optimized out; use `-O0` for full debuggability.

## Running Programs

- `run [args]` (r) -- start the program, optionally with arguments
- `start` -- run and break at `main()`
- `attach pid` -- attach to a running process
- `detach` -- release the inferior
- `kill` -- terminate the inferior

Control the environment: `set args arg1 arg2`, `set environment VAR=val`, `unset environment VAR`. Redirect I/O: `tty /dev/pts/N`. Change working directory: `cd dir`. Disable ASLR for reproducible addresses: `set disable-randomization on`.

## Multi-Process and Multi-Threaded Debugging

### Inferiors (processes)

Each debugged process is an "inferior" with its own address space, threads, and program. Commands: `info inferiors`, `inferior N` (switch), `add-inferior`, `clone-inferior`, `remove-inferiors N`.

### Threads

- `info threads` -- list all threads with their states
- `thread N` -- switch to thread N
- `thread apply all bt` -- backtrace every thread
- `thread name NAME` -- label the current thread
- `thread find REGEXP` -- search by name, target ID, or extra info
- `set scheduler-locking off|on|step` -- control which threads run during stepping (`on` freezes all others, `step` freezes during single-step only)
- `$_thread` -- convenience variable holding current thread number

### Fork debugging

- `set follow-fork-mode parent|child` -- which process to follow after fork
- `set detach-on-fork on|off` -- `off` keeps both parent and child under GDB
- `catch fork`, `catch vfork`, `catch exec` -- stop on process creation/exec

## Breakpoints

### Software breakpoints

- `break location [if cond]` (b) -- location can be a function name, `file:line`, `*address`, or `+offset`
- `tbreak location` -- temporary breakpoint, deleted after first hit
- `rbreak regex` -- set breakpoints on all functions matching a regular expression
- `set breakpoint pending on|off|auto` -- control behavior for unresolved symbols (e.g., in not-yet-loaded shared libraries)

### Hardware breakpoints

- `hbreak location` -- uses processor debug registers; required for ROM/flash code or when software breakpoints cannot be inserted

### Watchpoints (data breakpoints)

- `watch expr` -- stop when the value of `expr` changes (write watchpoint)
- `rwatch expr` -- stop when `expr` is read
- `awatch expr` -- stop on any read or write
- Hardware watchpoints are fast but limited by the number of debug registers (typically 4). Software watchpoints single-step the program and are much slower.
- `watch -location expr` -- watch the memory address that `expr` currently refers to, not the expression itself

### Catchpoints

- `catch syscall [name|number]` -- stop on system call entry/exit
- `catch signal [sig]` -- stop when a signal is delivered
- `catch throw` / `catch catch` -- C++ exception throw/catch
- `catch load [regexp]` / `catch unload [regexp]` -- shared library events

### Management

- `info breakpoints` (i b) -- list all breakpoints, watchpoints, catchpoints
- `delete [N]` (d) -- delete breakpoint N (all if no argument)
- `disable [N]` / `enable [N]` -- toggle without deleting
- `condition N expr` -- set or change the condition on breakpoint N
- `ignore N count` -- skip the next `count` hits
- `commands [N]` ... `end` -- attach commands to a breakpoint; use `silent` + `printf` + `continue` for printf-style debugging without stopping
- `save breakpoints file` -- persist breakpoints; reload with `source file`

### Dynamic Printf

- `dprintf location,format,args...` -- insert a printf at `location` without recompiling
- Styles: `set dprintf-style gdb` (GDB's own printf), `call` (call the inferior's printf), `agent` (execute on a remote agent)

## Stepping and Continuing

- `continue [ignore-count]` (c) -- resume execution
- `step [count]` (s) -- step into functions, by source line
- `next [count]` (n) -- step over function calls
- `finish` (fin) -- run until the current function returns
- `until [location]` (u) -- run forward past the current line; useful for exiting loops
- `advance location` -- continue to `location` or until the current frame exits
- `stepi` (si) / `nexti` (ni) -- single machine instruction
- `skip function NAME` / `skip file FILE` -- skip uninteresting code during stepping
- `set step-mode on` -- step into functions that lack debug info (normally skipped)

## Signal Handling

- `handle signal keywords` -- keywords: `stop`/`nostop`, `print`/`noprint`, `pass`/`nopass` (whether to deliver to the inferior)
- `info signals` -- show the disposition table for all signals
- `signal 0` -- resume execution suppressing the currently pending signal
- `$_siginfo` -- access the `siginfo_t` structure (e.g., faulting address for SIGSEGV)

## Reverse Execution

Requires recording: `record full` (software recording) or `record btrace` (Intel Processor Trace, lower overhead).

- `reverse-continue` (rc) -- run backward to the previous stop event
- `reverse-step` / `reverse-next` -- step backward by source line
- `reverse-finish` -- run backward until entry of the current function
- `reverse-stepi` / `reverse-nexti` -- single instruction backward
- `set exec-direction reverse|forward` -- make `step`/`next`/`continue` go backward
- `record save filename` / `record restore filename` -- persist the execution log
- `record delete` -- discard the recorded future and begin recording anew from the current point

## Tracepoints

Tracepoints collect data without stopping the program, suitable for production-like environments.

- `trace location` / `ftrace location` (fast tracepoint, uses jump pad) / `strace location` (static marker)
- `actions` block: `collect expr` (`$regs`, `$args`, `$locals`), `teval expr`, `while-stepping N`
- `tvariable $name [= val]` -- persistent trace state variable across hits
- `tstart` / `tstop` / `tstatus` -- control and query tracing
- `tfind mode` -- navigate collected frames; `tdump` -- display current trace frame
- `tsave filename` -- save trace data to a file
- `set disconnected-tracing on` -- tracing continues after GDB disconnects
- `set circular-trace-buffer on` -- overwrite oldest frames when the buffer is full

## Stack Examination

- `backtrace [N]` (bt) -- print the top N frames; `bt full` includes local variables; `bt -N` prints the bottom N frames
- `frame N` (f) / `up [N]` / `down [N]` -- select a stack frame
- `info frame` -- verbose details: saved registers, frame addresses, calling convention
- `info args` -- function arguments in the selected frame
- `info locals` -- local variables in the selected frame

## Source Examination

- `list [linespec]` (l) / `list first,last` / `list -` -- display source lines
- `directory dir` -- add a directory to the source search path
- `set substitute-path from to` -- remap source file locations (e.g., after moving the build tree)
- `disassemble [/m] [/r] [addr]` -- show machine code; `/m` interleaves source, `/r` shows raw bytes
- `set disassembly-flavor att|intel` -- choose syntax style

## Data Examination

### print command

- `print [/fmt] expr` (p) -- evaluate and display an expression
- Formats: `x` (hex), `d` (decimal), `u` (unsigned), `o` (octal), `t` (binary), `a` (address), `c` (char), `f` (float), `s` (string), `r` (raw, no pretty-printer)
- `print *array@len` -- display `len` elements starting at pointer `array`

### x (examine memory)

- `x/Nfu addr` -- N = repeat count, f = format, u = unit size (`b` byte, `h` halfword, `w` word, `g` giant/8 bytes)
- `x/16xw $sp` -- 16 words in hex from the stack pointer
- `x/10i $pc` -- 10 instructions from the program counter

### Automatic display

- `display [/fmt] expr` -- re-evaluate and print at every stop
- `display/i $pc` -- always show the next instruction
- `undisplay N` -- remove an auto-display entry

### Other data commands

- Value history: `$1`, `$2`, `$$` (previous), `$$N` (N-th previous)
- Convenience variables: `set $i = 0`, then `p arr[$i++]`; press Enter to repeat
- `info registers` / `info all-registers` -- dump register contents
- `ptype /o struct_name` -- show struct layout with field offsets, sizes, and padding holes

## Checkpoints

- `checkpoint` -- snapshot the full program state (memory, registers, file descriptors); GNU/Linux only, uses fork
- `restart checkpoint-id` -- restore to a saved checkpoint
- `info checkpoints` / `delete checkpoint id`
- Checkpoints preserve the address layout, avoiding ASLR differences between runs

## Text User Interface (TUI)

- `tui enable` / `Ctrl-x a` -- toggle the split-screen curses interface
- Windows: `src` (source), `asm` (assembly), `regs` (registers), `cmd` (command)
- `layout src|asm|split|regs` -- choose a window arrangement
- `focus src|asm|cmd|regs` -- direct keyboard input to a window
- SingleKey mode (`Ctrl-x s`): single-key shortcuts -- `c` = continue, `n` = next, `s` = step, `f` = finish, `r` = run, `u` = up, `d` = down

## Python Scripting

GDB embeds a Python 2/3 interpreter for automation and extension.

- `python` ... `end` -- execute a Python block; `python-interactive` (pi) -- start a REPL
- Core API: `gdb.execute(cmd)`, `gdb.parse_and_eval(expr)`, `gdb.breakpoints()`, `gdb.selected_frame()`, `gdb.selected_thread()`
- `gdb.Value` -- wraps inferior values; supports `cast()`, `dereference()`, `.type`, subscript, and arithmetic
- `gdb.Type` -- represents types; `.fields()`, `.sizeof`, `TYPE_CODE_STRUCT`, `TYPE_CODE_PTR`, etc.
- `gdb.Breakpoint` -- scriptable breakpoints; override `stop(self)` to return `True`/`False`
- `gdb.Command` -- define custom commands with `invoke(self, arg, from_tty)`
- `gdb.Parameter` -- expose custom `set`/`show` settings
- Pretty-printing: implement `to_string()`, `children()`, `display_hint()` on a printer class
- Events: `gdb.events.stop`, `.cont`, `.exited`, `.new_objfile` -- register callbacks
- Frame unwinders, frame filters, and xmethods for advanced C++ debugging
- Auto-loading: place `OBJFILE-gdb.py` alongside shared libraries; controlled by `set auto-load safe-path`

## See also

- [gdb-remote-debugging](gdb-remote-debugging.md)
- [gnu-toolchain](gnu-toolchain.md)
- [concept-reverse-debugging](../concepts/concept-reverse-debugging.md)
