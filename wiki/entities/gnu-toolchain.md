---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [advanced-linux-programming, debugging-with-gdb]
tags: [gcc, gdb, make, development-tools, toolchain]
---

# GNU Development Toolchain

The GNU toolchain is the standard development environment for Linux kernel and application development. It comprises the compiler (GCC), build system (Make), debugger (GDB), and supporting tools for profiling, binary analysis, and memory debugging.

## GCC (GNU Compiler Collection)

GCC compiles C, C++, and other languages through a four-stage pipeline:

1. **Preprocessing** (`cpp`) — macro expansion, `#include` processing, conditional compilation.
2. **Compilation** — source-to-assembly translation with optimization.
3. **Assembly** (`as`) — assembly-to-object-code translation.
4. **Linking** (`ld`) — combines object files and libraries into an executable.

### Key Flags

| Category | Flags | Purpose |
|----------|-------|---------|
| Warnings | `-Wall`, `-Wextra`, `-Werror`, `-pedantic` | Enable comprehensive warnings; treat as errors |
| Optimization | `-O0` (none), `-O1`, `-O2` (standard), `-O3` (aggressive), `-Os` (size) | Trade compile time for runtime performance |
| Debugging | `-g`, `-ggdb` | Include debug symbols for GDB |
| Preprocessing | `-D NAME=VALUE`, `-I dir` | Define macros, add include paths |
| Linking | `-l library`, `-L dir`, `-static`, `-shared` | Link libraries, set library paths |
| Code generation | `-fPIC` (position-independent), `-march=native` | Control generated code properties |

### Libraries

- **Static libraries** (`.a`) — created with `ar rcs libfoo.a obj1.o obj2.o`. Object code is copied into the executable at link time. Larger binaries, no runtime dependency.
- **Shared libraries** (`.so`) — created with `gcc -shared -fPIC -o libfoo.so obj1.o obj2.o`. Loaded at runtime by the dynamic linker (`ld-linux.so`). Smaller binaries, shared across processes. Versioning via sonames (`libfoo.so.1`).

## GNU Make

Dependency-driven build automation. A `Makefile` declares targets, prerequisites, and recipes:

```makefile
target: prerequisites
	recipe commands (must be tab-indented)
```

### Key Features

- **Automatic variables**: `$@` (target), `$<` (first prerequisite), `$^` (all prerequisites).
- **Pattern rules**: `%.o: %.c` matches any `.c`-to-`.o` transformation.
- **Phony targets**: `.PHONY: clean all install` — targets that don't represent files.
- **Variables**: `CC = gcc`, `CFLAGS = -Wall -O2`, used as `$(CC) $(CFLAGS)`.
- **Parallel builds**: `make -j$(nproc)` builds independent targets concurrently.

## GDB (GNU Debugger)

Interactive source-level debugger for compiled programs:

| Command | Purpose |
|---------|---------|
| `break func` / `break file:line` | Set a breakpoint |
| `run [args]` | Start the program |
| `next` / `step` | Execute next line (over / into functions) |
| `continue` | Resume execution |
| `print expr` | Evaluate and display an expression |
| `backtrace` | Show call stack |
| `watch expr` | Break when expression changes (watchpoint) |
| `info threads` | List threads |
| `thread N` | Switch to thread N |
| `attach pid` | Attach to a running process |
| `core file` | Analyze a core dump |

Compile with `-g` (or `-ggdb` for GDB-specific extensions) and `-O0` for the best debugging experience.

For comprehensive GDB coverage — breakpoints, watchpoints, tracepoints, reverse debugging, Python scripting, GDB/MI, and TUI — see [gdb-debugger](gdb-debugger.md). For remote debugging with gdbserver and the Remote Serial Protocol, see [gdb-remote-debugging](gdb-remote-debugging.md).

## Supporting Tools

| Tool | Purpose |
|------|---------|
| `strace` | Trace system calls — shows every syscall with arguments and return values |
| `ltrace` | Trace library calls — shows calls to shared library functions |
| `gprof` | Flat + call-graph CPU profiling (compile with `-pg`) |
| `gcov` | Line-by-line code coverage analysis (compile with `--coverage`) |
| `valgrind` | Memory error detection (uninitialized reads, use-after-free, leaks) — no recompilation needed |
| `objdump` | Disassemble object files and executables |
| `nm` | List symbols in object files |
| `ldd` | List shared library dependencies of an executable |
| `dmalloc` / `Electric Fence` | Debug memory allocators with boundary checking |

## Relationship to the Kernel

The kernel is built exclusively with GCC (and now also Clang) using the [kernel-build-system](kernel-build-system.md) (Kbuild), which is a sophisticated layer atop GNU Make. GCC's inline assembly support ([concept-inline-assembly](../concepts/concept-inline-assembly.md)) is essential for kernel development. GDB can debug the kernel itself via KGDB (kernel GDB stub) or with QEMU/KVM.

## See also

- [gdb-debugger](gdb-debugger.md)
- [gdb-remote-debugging](gdb-remote-debugging.md)
- [concept-reverse-debugging](../concepts/concept-reverse-debugging.md)
- [kernel-build-system](kernel-build-system.md)
- [kernel-configuration](kernel-configuration.md)
- [concept-inline-assembly](../concepts/concept-inline-assembly.md)
