---
type: entity
created: 2026-04-08
updated: 2026-04-08
sources: [understanding-the-linux-kernel, linux-kernel-in-a-nutshell]
tags: [modules, lkm, insmod, modprobe, symbol-export, modules-install]
---

# Kernel Modules

Loadable Kernel Modules (LKMs) are the Linux kernel's mechanism for extending a monolithic kernel at runtime without recompiling or rebooting. They allow device drivers, filesystems, network protocols, and other kernel features to be loaded on demand and unloaded when no longer needed. This gives Linux the flexibility typically associated with microkernel architectures while retaining the performance advantages of a monolithic design.

## Module Architecture

### The module Structure

Every loaded module is represented by a `struct module` instance. Key fields include:

| Field | Purpose |
|-------|---------|
| `state` | Current lifecycle state: `MODULE_STATE_COMING` (being loaded), `MODULE_STATE_LIVE` (active), or `MODULE_STATE_GOING` (being removed) |
| `name` | The module's string name (e.g., "ext3", "e1000") |
| `syms` / `num_syms` | Exported symbol table and count |
| `gpl_syms` / `num_gpl_syms` | GPL-restricted exported symbol table and count |
| `init` | Pointer to the module's initialization function |
| `exit` | Pointer to the module's cleanup function |
| `module_core` | Pointer to the module's core memory region (code + data that persists) |
| `module_init` | Pointer to the init-only memory region (freed after initialization) |
| `args` | Module parameter descriptions |
| `list` | Linked list node connecting all loaded modules |
| `modules_which_use_me` | List of `module_use` descriptors tracking which other modules depend on this one |
| `waiter` | Task waiting for module state transitions |
| `ref` | Per-CPU reference counters (one counter per CPU to avoid cache-line bouncing on SMP systems) |

### Module States

A module transitions through three states during its lifecycle:

```
MODULE_STATE_COMING  -->  MODULE_STATE_LIVE  -->  MODULE_STATE_GOING
   (loading)               (operational)           (unloading)
```

The state field is protected to ensure that operations like symbol resolution and reference counting behave correctly during transitions. For example, a module in `MODULE_STATE_COMING` state cannot yet be used by other modules for symbol resolution, and a module in `MODULE_STATE_GOING` state will refuse new reference count increments.

### Reference Counting

Module reference counts use **per-CPU counters** to avoid SMP scalability bottlenecks. Rather than a single atomic counter that would cause cache-line bouncing across CPUs, each CPU maintains its own counter. The total reference count is the sum of all per-CPU values. Functions that use module symbols call `try_module_get()` to increment the count and `module_put()` to decrement it. A module can only be unloaded when the aggregate reference count is zero.

## Loading a Module

### User-Space: insmod

The `insmod` utility performs the user-space portion of module loading:

1. **Reads the ELF object file** from disk into a user-space buffer. Kernel modules are compiled as relocatable ELF (Executable and Linkable Format) object files (`.ko` extension in 2.6, `.o` in 2.4).
2. **Invokes `sys_init_module()`**, passing the module image and any parameter strings to the kernel.

### Kernel-Space: sys_init_module()

The `sys_init_module()` system call performs the heavy lifting:

1. **Permission check**: Verifies that the caller has `CAP_SYS_MODULE` capability. Only privileged processes can load modules.

2. **Copy from user space**: Copies the ELF module image from user memory into a temporary kernel buffer using `copy_from_user()`.

3. **ELF validation**: Parses and validates the ELF headers — checks the magic number, verifies it is a relocatable object file (type `ET_REL`), and confirms the architecture matches the running kernel.

4. **Section processing**: Iterates through the ELF section headers. Key sections include:
   - `.text` — executable code
   - `.data` / `.rodata` — initialized data and read-only data
   - `.bss` — uninitialized data
   - `__versions` — CRC version information for symbol versioning (CONFIG_MODVERSIONS)
   - `.modinfo` — module metadata (license, author, description, parameter definitions)
   - `__ksymtab` / `__ksymtab_gpl` — exported symbol tables

5. **Memory allocation**: Allocates kernel memory for the module. Two separate regions are allocated:
   - **Core section** (`module_core`): Code and data that persist for the module's entire lifetime.
   - **Init section** (`module_init`): Code and data needed only during initialization; freed afterward to reclaim memory.

6. **Relocation**: Processes ELF relocation entries to fix up addresses. Since the module was compiled as a relocatable object, all absolute addresses must be adjusted to reflect the actual kernel addresses where the module was loaded. This includes resolving references to kernel symbols and to symbols from other loaded modules.

7. **Symbol resolution**: Resolves undefined symbols in the module against:
   - The kernel's exported symbol table
   - Symbol tables of other currently loaded modules
   - If CONFIG_MODVERSIONS is enabled, CRC checksums are compared to detect ABI mismatches between the module and the kernel it was compiled for.

8. **Parameter setup**: Processes module parameters passed from the command line, matching them against parameter descriptors defined in the module via `module_param()`.

9. **Calls init()**: Invokes the module's initialization function (the function registered via `module_init()`). This is where the module registers its functionality — device drivers call `register_chrdev()` or `pci_register_driver()`, filesystems call `register_filesystem()`, etc.

10. **Frees init memory**: After the init function returns successfully, the `module_init` memory region is freed. The module's state transitions from `MODULE_STATE_COMING` to `MODULE_STATE_LIVE`.

11. **Failure handling**: If any step fails, allocated memory is freed, partial registrations are undone, and an appropriate error code is returned to user space.

## Unloading a Module

### User-Space: rmmod

The `rmmod` utility invokes the `sys_delete_module()` system call with the module name.

### Kernel-Space: sys_delete_module()

1. **Permission check**: Verifies `CAP_SYS_MODULE` capability.

2. **Finds the module**: Looks up the module by name in the global module list.

3. **State check**: Verifies the module is in `MODULE_STATE_LIVE` state. If it is already in `MODULE_STATE_GOING`, another unload is in progress.

4. **Dependency check**: Examines the `modules_which_use_me` list. If any other loaded module depends on this one, unloading is refused with `-EWOULDBLOCK`. The dependent modules must be unloaded first.

5. **Reference count check**: Sums the per-CPU reference counters. If the total is non-zero, the module is still in use and cannot be unloaded. The behavior when the reference count is non-zero depends on the flags:
   - Default: return `-EWOULDBLOCK` immediately.
   - `O_NONBLOCK` flag absent and `O_TRUNC` absent: wait for the reference count to reach zero.
   - Forced removal (if CONFIG_MODULE_FORCE_UNLOAD is enabled): proceed despite non-zero reference count (dangerous — can cause crashes).

6. **State transition**: Sets the module state to `MODULE_STATE_GOING` to prevent new users.

7. **Calls exit()**: Invokes the module's cleanup function (registered via `module_exit()`). This function must unregister everything that `init()` registered — device drivers, filesystems, interrupt handlers, procfs entries, etc.

8. **Frees memory**: Releases the module's core memory region and the `struct module` itself. Removes the module from the global module list.

## Symbol Export

Modules interact with the kernel and with each other through **exported symbols**. The kernel maintains two symbol tables:

### EXPORT_SYMBOL()

```c
EXPORT_SYMBOL(symbol_name);
```

Makes the symbol (function or variable) available to **all** loadable modules, regardless of their license. Exported symbols are placed in the `__ksymtab` section.

### EXPORT_SYMBOL_GPL()

```c
EXPORT_SYMBOL_GPL(symbol_name);
```

Makes the symbol available **only** to modules that declare a GPL-compatible license via `MODULE_LICENSE()`. These symbols are placed in the `__ksymtab_gpl` section. This mechanism was introduced to ensure that certain internal kernel APIs — particularly newer interfaces where the kernel developers want to prevent proprietary drivers from using them — are restricted to free software modules.

### Symbol Versioning (CONFIG_MODVERSIONS)

When enabled, each exported symbol is tagged with a CRC checksum computed from the symbol's function prototype (or variable type). When a module is loaded, the kernel compares the CRC of each symbol the module uses against the CRC of the currently exported symbol. A mismatch indicates that the module was compiled against a different kernel version where the function signature or data structure layout differed, and loading is refused. This prevents subtle data corruption or crashes from ABI incompatibilities.

## MODULE_LICENSE() and Kernel Tainting

Every module should declare its license:

```c
MODULE_LICENSE("GPL");
MODULE_LICENSE("GPL v2");
MODULE_LICENSE("GPL and additional rights");
MODULE_LICENSE("Dual BSD/GPL");
MODULE_LICENSE("Dual MIT/GPL");
MODULE_LICENSE("Proprietary");
```

If a module is loaded without a recognized GPL-compatible license, the kernel marks itself as **tainted**. The taint flag is visible in `/proc/sys/kernel/tainted` and is included in oops and panic reports. Kernel developers use the taint flag to quickly determine whether a bug report involves proprietary modules — tainted kernels receive lower priority in community bug triage because the proprietary module may be the cause of the problem.

Taint flags include:
- **P** — Proprietary module loaded
- **F** — Module was force-loaded (bypassing version checks)
- **S** — SMP kernel running on hardware not certified for SMP
- **R** — Module was force-unloaded
- **M** — Machine check exception occurred
- **B** — Bad page referenced

## Dependency Tracking

### modules_which_use_me

Each `struct module` contains a list head `modules_which_use_me` that links `module_use` descriptors. When module A resolves a symbol exported by module B during loading, a `module_use` descriptor is created recording that A depends on B. This descriptor is added to B's `modules_which_use_me` list.

The `module_use` structure contains:

```c
struct module_use {
    struct list_head source_list;  /* added to using module's list */
    struct list_head target_list;  /* added to used module's list */
    struct module *source;         /* module that uses the symbol */
    struct module *target;         /* module that provides the symbol */
};
```

This bidirectional linkage allows the kernel to efficiently answer two questions:
- "Which modules does module A depend on?" (needed when unloading A — must decrement ref counts)
- "Which modules depend on module B?" (needed when unloading B — must refuse if the list is non-empty)

## Demand Loading with request_module()

Kernel code can trigger automatic module loading by calling **`request_module()`**. This is used in situations where the kernel encounters a need for functionality that might be available as a module — for example, when a process tries to create a socket with an unknown protocol family, or when a device with an unrecognized major number is accessed.

### Mechanism

1. `request_module()` spawns a kernel thread that runs **`/sbin/modprobe`** as a user-space process.
2. `modprobe` reads **`/lib/modules/<version>/modules.dep`**, a file generated by `depmod` that records the full dependency graph of all available modules.
3. `modprobe` determines the transitive closure of dependencies — if module A requires B, and B requires C, all three must be loaded in the correct order (C first, then B, then A).
4. `modprobe` calls `sys_init_module()` for each module in dependency order.
5. `request_module()` returns to the caller, which retries the operation that triggered the module request.

### Module Aliases

`modprobe` also supports **aliases**, which map abstract identifiers to module names. For instance, `char-major-10-135` might map to the `rtc` module. Aliases are stored in `modules.alias` (generated by `depmod` from MODULE_ALIAS() declarations in module source code) and in `/etc/modprobe.conf`.

## sysfs Interface

Each loaded module is represented in the **sysfs** filesystem under `/sys/module/<name>/`. The entries include:

| Path | Content |
|------|---------|
| `/sys/module/<name>/` | Directory for the module |
| `/sys/module/<name>/parameters/` | Directory containing one file per module parameter, readable and (if writable) writable at runtime |
| `/sys/module/<name>/sections/` | Directory showing the kernel addresses of the module's ELF sections (useful for debugging with gdb) |
| `/sys/module/<name>/refcnt` | Current aggregate reference count |
| `/sys/module/<name>/srcversion` | Source version hash |

Module parameters are declared in source code with `module_param()`:

```c
static int debug = 0;
module_param(debug, int, 0644);
MODULE_PARM_DESC(debug, "Enable debug logging");
```

The third argument (`0644`) sets the sysfs file permissions — `0644` means the parameter is readable by all and writable by root at runtime via `/sys/module/<name>/parameters/debug`.

## What Cannot Be a Module

Not all kernel functionality can be implemented as a loadable module. The following categories must be compiled directly into the kernel (built-in):

### Core Data Structure Modifications

Any code that modifies the layout of fundamental kernel data structures such as `task_struct` (the [process descriptor](process-management.md)) cannot be a module. If a module added a field to `task_struct`, all existing code that accesses the structure would see incorrect offsets, causing data corruption. The structure layout must be fixed at compile time.

### Fundamental Memory Allocators

The **[buddy system](memory-management.md)** page allocator and the basic **[slab allocator](memory-management.md)** infrastructure cannot be modules because they are needed from the earliest moments of kernel initialization — long before the module loading infrastructure is available. `kmalloc()`, `__get_free_pages()`, and related functions must be available from `start_kernel()` onward.

### Core Scheduling and Synchronization

The [process-scheduler](process-scheduler.md) itself and the fundamental synchronization primitives ([spinlocks](../concepts/concept-kernel-synchronization.md), [semaphores](../concepts/concept-kernel-synchronization.md), [RCU](../concepts/concept-kernel-synchronization.md)) are compiled in. These are used by every part of the kernel, including the module loading code itself — a circular dependency would make them impossible to load as modules.

### Early Boot Infrastructure

Code needed during the [boot-process](boot-process.md) before `do_basic_setup()` completes — including the architecture setup code, early console, boot memory allocator, and the module loading infrastructure itself — must be built in.

### Interrupt and Exception Handling

The core [interrupt-handling](interrupt-handling.md) framework, including the IDT setup, exception handlers (page fault, general protection fault), and the basic IRQ dispatch mechanism, must be built in. Individual device interrupt handlers (e.g., for a specific network card) can be in modules, but the dispatch infrastructure cannot.

### Critical Filesystem and Block Layers

If the root filesystem's driver or the block device driver for the boot disk were modules, they could not be loaded because the module files reside on those very filesystems. This chicken-and-egg problem is traditionally solved with **initrd** or **initramfs**, which provide a temporary root filesystem in RAM containing the needed modules. However, the code to mount initrd/initramfs itself must be built in.

## Module Installation

After building the kernel, `make modules_install` installs all compiled modules to `/lib/modules/<version>/`. This directory contains:

| Path | Content |
|------|---------|
| `kernel/` | Module files (`.ko`) organized by subsystem (`drivers/`, `fs/`, `net/`, `sound/`, etc.) |
| `modules.dep` | Dependency graph generated by `depmod` -- maps each module to its dependencies |
| `modules.alias` | Device-to-module mapping from `MODULE_DEVICE_TABLE()` and `MODULE_ALIAS()` |
| `modules.symbols` | Exported-symbol-to-module mapping |
| `modules.pcimap` | PCI vendor/device ID to module mapping (used by hotplug) |
| `modules.usbmap` | USB vendor/product ID to module mapping |
| `build` | Symlink to the kernel build directory (used for out-of-tree module builds) |
| `source` | Symlink to the kernel source directory |

Running `depmod -a` regenerates the dependency and map files. This is called automatically by `make modules_install`. Use `INSTALL_MOD_PATH=/mnt/target` to install to an alternate root.

## Module Parameters at Boot Time

Module parameters can be passed at boot time via the kernel command line for built-in modules, or via `modprobe` arguments for loadable modules:

```sh
# Boot command line (for built-in modules):
module_name.parameter=value

# modprobe (for loadable modules):
modprobe module_name parameter=value

# /etc/modprobe.conf or /etc/modprobe.d/*.conf (persistent):
options module_name parameter=value
```

Parameters declared with `module_param()` appear as files under `/sys/module/<name>/parameters/`, allowing runtime inspection and (if permissions allow) modification.

## See also

- [boot-process](boot-process.md)
- [kernel-build-system](kernel-build-system.md)
- [kernel-configuration](kernel-configuration.md)
- [process-management](process-management.md)
- [memory-management](memory-management.md)
- [virtual-filesystem](virtual-filesystem.md)
- [interrupt-handling](interrupt-handling.md)
- [concept-kernel-synchronization](../concepts/concept-kernel-synchronization.md)
