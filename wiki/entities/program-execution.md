---
type: entity
created: 2026-04-08
updated: 2026-04-08
sources: [understanding-the-linux-kernel]
tags: [program-execution, elf, execve, dynamic-linking]
---

# Program Execution

Program execution in Linux begins with the `execve()` system call and involves parsing the executable format, mapping segments into memory, setting up the user-mode stack, and transferring control to the program's entry point (or to the dynamic linker for shared-library programs). The kernel supports multiple executable formats through a pluggable binary handler framework.

## The execve() System Call

`execve()` replaces the current process's program with a new one. Its signature is:

```c
int execve(const char *filename, char *const argv[], char *const envp[]);
```

The kernel implementation follows this path:

```
sys_execve()
  -> do_execve(filename, argv, envp, regs)
    -> open_exec(filename)                   /* open the executable file */
    -> prepare linux_binprm                  /* build the binary parameter block */
    -> copy_strings(argc, argv, bprm)        /* copy argv from user space */
    -> copy_strings(envc, envp, bprm)        /* copy envp from user space */
    -> search_binary_handler(bprm, regs)     /* find and invoke the right loader */
```

### linux_binprm

The `linux_binprm` structure collects all information needed to load and execute a program:

| Field | Purpose |
|-------|---------|
| `buf[BINPRM_BUF_SIZE]` | First 128 bytes of the file — used to detect the format (ELF magic, `#!` shebang, etc.) |
| `filename` | Path to the executable |
| `interp` | Path to the interpreter (initially same as `filename`, updated for scripts) |
| `page[]` | Pages holding the argv and envp strings (copied from user space) |
| `argc`, `envc` | Argument and environment variable counts |
| `e_uid`, `e_gid` | Effective UID/GID (reflecting setuid/setgid bits) |
| `file` | The `struct file` of the opened executable |
| `cred_prepared` | Security credentials computed from the executable |
| `mm` | New `mm_struct` allocated for the program |

### search_binary_handler()

The kernel maintains a linked list of `linux_binfmt` structures, each representing a supported executable format. `search_binary_handler()` iterates this list, calling each format's `load_binary()` function until one succeeds:

| Format | Module | Magic | Handler |
|--------|--------|-------|---------|
| **ELF** | `binfmt_elf` | `\x7fELF` | `load_elf_binary()` |
| **Script** | `binfmt_script` | `#!` | `load_script()` |
| **a.out** | `binfmt_aout` | Varies | `load_aout_binary()` (legacy) |
| **Flat** | `binfmt_flat` | — | For MMU-less embedded systems |
| **Misc** | `binfmt_misc` | Configurable | User-registered formats (via `/proc/sys/fs/binfmt_misc`) |

If no handler recognizes the format, `execve()` returns `-ENOEXEC`.

## ELF Format

The **Executable and Linkable Format** (ELF) is the standard binary format on Linux. An ELF file has a dual view: **program headers** (used at load time) and **section headers** (used at link time).

### ELF Header

At offset 0 of every ELF file:

| Field | Purpose |
|-------|---------|
| `e_ident[16]` | Magic number (`\x7fELF`), class (32/64-bit), endianness, OS/ABI |
| `e_type` | Object type: `ET_EXEC` (static executable), `ET_DYN` (shared object / PIE), `ET_REL` (relocatable) |
| `e_machine` | Target architecture (e.g., `EM_386`, `EM_X86_64`) |
| `e_entry` | **Entry point** — virtual address where execution begins |
| `e_phoff` | Offset of the program header table |
| `e_shoff` | Offset of the section header table |
| `e_phnum` | Number of program headers |
| `e_shnum` | Number of section headers |

### Program Headers (Segments)

Program headers describe **segments** — contiguous regions of the file that should be mapped into memory at load time:

| Type | Purpose |
|------|---------|
| `PT_LOAD` | Loadable segment — mapped into memory via `do_mmap()`. Typically there are two: one for text (read + execute) and one for data (read + write). |
| `PT_INTERP` | Contains the pathname of the dynamic linker (e.g., `/lib/ld-linux.so.2`). Present only in dynamically linked executables. |
| `PT_PHDR` | Describes the program header table itself. The dynamic linker uses this to find the program headers in memory. |
| `PT_DYNAMIC` | Points to the `.dynamic` section containing dynamic linking information (needed shared libraries, symbol tables, relocation entries). |
| `PT_NOTE` | Auxiliary information (build ID, ABI tags). |
| `PT_GNU_STACK` | Indicates whether the stack should be executable. |

Each `PT_LOAD` header specifies:
- `p_vaddr` — virtual address to load at
- `p_memsz` — size in memory (may be larger than file size for BSS)
- `p_filesz` — size in the file
- `p_flags` — permissions (`PF_R`, `PF_W`, `PF_X`)

### Sections

Sections are a link-time view. Key sections in a typical executable:

| Section | Content |
|---------|---------|
| `.text` | Executable code |
| `.rodata` | Read-only data (string literals, constants) |
| `.data` | Initialized global/static variables |
| `.bss` | Uninitialized global/static variables (occupies no file space — zeroed at load time) |
| `.symtab` / `.dynsym` | Symbol tables (static / dynamic) |
| `.strtab` / `.dynstr` | String tables for symbol names |
| `.rel.dyn` / `.rel.plt` | Relocation entries for dynamic linking |
| `.plt` / `.got` / `.got.plt` | Procedure Linkage Table and Global Offset Table for lazy symbol resolution |
| `.init` / `.fini` | Initialization and finalization code |

## load_elf_binary()

The ELF loader in `binfmt_elf.c` performs the following steps:

### 1. Validate the ELF Header

Check the magic number (`\x7fELF`), class, endianness, and architecture. Verify `e_type` is `ET_EXEC` or `ET_DYN`.

### 2. Read Program Headers

Parse the program header table to find `PT_LOAD`, `PT_INTERP`, and other segment descriptors.

### 3. Tear Down the Old Address Space

Call `flush_old_exec()`, which:
- Calls `exec_mmap()` to replace the current `mm_struct` with a new, empty one. The old `mm_struct` (and all its VMAs, page tables, and page frames) is released.
- Clears pending signals.
- Closes file descriptors marked `FD_CLOEXEC`.
- Resets signal handlers to `SIG_DFL`.
- Flushes the instruction cache.

This is the **point of no return** — after `flush_old_exec()`, the old program's state is gone and `execve()` can no longer fail and return to the caller.

### 4. Map PT_LOAD Segments

For each `PT_LOAD` segment, call `do_mmap()` (via `elf_map()`) to create a VMA:

- **Text segment**: mapped with `PROT_READ | PROT_EXEC`, backed by the ELF file (demand-paged from disk).
- **Data segment**: mapped with `PROT_READ | PROT_WRITE`, also backed by the file.
- **BSS**: the portion of the data segment where `p_memsz > p_filesz` is zero-filled. The kernel calls `do_brk()` to create an anonymous mapping for the BSS region and the initial heap.

### 5. Set Up the BSS and Heap

`do_brk()` creates a zero-filled anonymous VMA for the BSS. The initial `brk` (heap boundary) is set to the end of the BSS. Future `brk()` / `malloc()` calls extend this region.

### 6. Load the Dynamic Linker (if needed)

If a `PT_INTERP` segment was found:

1. Open the interpreter file (e.g., `/lib/ld-linux.so.2`).
2. Read its ELF header and program headers.
3. Map the interpreter's `PT_LOAD` segments into the process's address space via `do_mmap()`.
4. Set the entry point to the **interpreter's entry point** (not the program's `e_entry`). The interpreter will eventually jump to the program's `e_entry` after resolving shared libraries.

### 7. Set Up the User-Mode Stack

The kernel prepares the user-mode stack (at the top of the user address space, growing downward) with the following layout:

```
High addresses
+---------------------------+
| envp strings              |  (null-terminated strings)
| argv strings              |  (null-terminated strings)
| padding / alignment       |
+---------------------------+
| auxiliary vector (auxv)   |  (AT_* entries, NULL-terminated)
| NULL                      |  (end of envp)
| envp[envc-1]              |  (pointers to env strings)
| ...                       |
| envp[0]                   |
| NULL                      |  (end of argv)
| argv[argc-1]              |  (pointers to arg strings)
| ...                       |
| argv[0]                   |
| argc                      |  (integer)
+---------------------------+  <-- initial stack pointer (esp)
Low addresses (stack grows down)
```

The **auxiliary vector** (`auxv`) provides information from the kernel to the dynamic linker and C library:

| Entry | Purpose |
|-------|---------|
| `AT_PHDR` | Address of program headers in memory |
| `AT_PHENT` | Size of each program header entry |
| `AT_PHNUM` | Number of program headers |
| `AT_ENTRY` | Program's entry point (`e_entry`) |
| `AT_BASE` | Base address where the interpreter was loaded |
| `AT_UID`, `AT_EUID`, `AT_GID`, `AT_EGID` | Process credentials |
| `AT_PAGESZ` | System page size (4096) |
| `AT_CLKTCK` | Clock ticks per second (`HZ`) |
| `AT_PLATFORM` | CPU architecture string |
| `AT_HWCAP` | CPU feature flags (SSE, MMX, etc.) |
| `AT_RANDOM` | Address of 16 random bytes (for stack canary seeding) |

### 8. Transfer Control

`start_thread(regs, entry, stack)` modifies the saved `pt_regs` on the kernel stack:
- Sets `eip` (instruction pointer) to the entry point — the dynamic linker's entry if dynamically linked, or `e_entry` if statically linked.
- Sets `esp` (stack pointer) to the top of the prepared user stack.
- Sets segment registers to user-mode values (`__USER_CS`, `__USER_DS`).

When the system call returns to user mode (via `ret_from_sys_call` / `iret`), the CPU loads these values and begins executing the new program.

## Dynamic Linking

For dynamically linked programs, the kernel does not resolve shared library dependencies — it delegates this to the **dynamic linker** (`ld-linux.so.2`), which is itself an ELF shared object mapped by the kernel.

### Dynamic Linker Startup

The dynamic linker's entry point (set by `start_thread()`) begins execution and:

1. **Finds itself**: uses the auxiliary vector (`AT_BASE`) to determine its own load address.
2. **Relocates itself**: applies relocations to its own GOT/PLT (bootstrap — the linker must be able to resolve its own symbols before it can resolve anything else).
3. **Reads the executable's `.dynamic` section**: finds the list of needed shared libraries (`DT_NEEDED` entries).
4. **Loads shared libraries**: for each needed library, searches the library path (`DT_RPATH`, `LD_LIBRARY_PATH`, `/etc/ld.so.cache`, default paths), opens the `.so` file, and maps its `PT_LOAD` segments via `mmap()`.
5. **Resolves symbols and applies relocations**: patches the GOT (Global Offset Table) entries so that function calls through the PLT (Procedure Linkage Table) reach the correct addresses.
6. **Calls initialization functions**: runs `.init` sections and `DT_INIT_ARRAY` constructors in dependency order.
7. **Jumps to the program's entry point** (`e_entry`, obtained from `AT_ENTRY` in the auxiliary vector).

### Lazy Binding

By default, PLT entries are resolved **lazily** — the first call to a function goes through the PLT stub, which invokes the dynamic linker to resolve the symbol and patch the GOT entry. Subsequent calls go directly through the patched GOT. This avoids the startup cost of resolving all symbols upfront. The environment variable `LD_BIND_NOW=1` (or the `DF_BIND_NOW` flag) forces eager resolution.

## Script Handling

When `search_binary_handler()` encounters a file beginning with `#!` (shebang), `load_script()` handles it:

1. Parse the first line to extract the interpreter path and optional argument (e.g., `#!/usr/bin/python -u`).
2. Update `bprm->interp` to point to the interpreter.
3. Shift the argument list: the script filename becomes `argv[1]` (or `argv[2]` if the shebang has an argument), and the interpreter becomes `argv[0]`.
4. Re-read the first 128 bytes of the **interpreter** binary into `bprm->buf`.
5. Call `search_binary_handler()` **recursively** to load the interpreter (which is typically an ELF binary).

This recursive invocation means the kernel handles `#!/usr/bin/env python` correctly: `load_script()` processes the shebang, then `load_elf_binary()` loads `/usr/bin/env`, which in turn execs `python`. The kernel limits recursion depth to prevent infinite loops (e.g., two scripts referencing each other as interpreters).

## flush_old_exec(): The Point of No Return

`flush_old_exec()` is called early in every `load_*_binary()` function and performs the irreversible destruction of the old program's state:

1. **de_thread()**: if the process is multi-threaded, kills all other threads in the thread group and waits for them to exit. The exec'ing thread becomes the thread group leader.
2. **exec_mmap()**: allocates a new `mm_struct`, installs it as the process's address space, and releases the old one. All old VMAs, page tables, and mapped pages are freed.
3. **flush_thread()**: resets architecture-specific state (debug registers, I/O permissions, etc.).
4. **Clear signal handlers**: all signal handlers are reset to `SIG_DFL` (signals with `SIG_IGN` remain ignored). Pending signals are cleared.
5. **Close FD_CLOEXEC files**: file descriptors marked with the close-on-exec flag are closed.

After `flush_old_exec()` returns, there is no program to "return to" if loading fails — this is why it is the point of no return. All subsequent steps in the loader must succeed.

## See also

- [process-management](process-management.md)
- [concept-memory-addressing](../concepts/concept-memory-addressing.md)
- [concept-copy-on-write](../concepts/concept-copy-on-write.md)
- [interrupt-handling](interrupt-handling.md)
