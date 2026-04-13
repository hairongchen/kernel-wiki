---
type: entity
created: 2026-04-08
updated: 2026-04-08
sources: [understanding-the-linux-kernel]
tags: [process-address-space, virtual-memory, page-fault, mmap]
---

# Process Address Space

Every Linux process has its own **virtual address space** -- a 4 GB linear address range on 32-bit x86, split into a 3 GB user portion (0x00000000 -- 0xBFFFFFFF) and a 1 GB kernel portion (0xC0000000 -- 0xFFFFFFFF). The kernel manages this address space through two key data structures: the **memory descriptor** (`mm_struct`) and **virtual memory areas** (`vm_area_struct`). Pages are populated lazily through the **page fault handler**, implementing demand paging, [concept-copy-on-write](../concepts/concept-copy-on-write.md), and swap-in.

## Memory descriptor: mm_struct

The `mm_struct` (aliased as `struct mm_struct`) is the top-level data structure describing a process's entire address space. It is pointed to by `task_struct->mm` (NULL for kernel threads, which use the previous process's page tables via `task_struct->active_mm`).

Key fields:

| Field | Description |
|-------|-------------|
| `pgd` | Pointer to the Page Global Directory (top-level page table). Loaded into the `cr3` register on context switch. |
| `mmap` | Head of the sorted linked list of all VMAs. |
| `mm_rb` | Root of the red-black tree organizing VMAs for fast lookup. |
| `mmap_cache` | The VMA returned by the most recent `find_vma()` call (a one-entry cache exploiting locality). |
| `map_count` | Number of VMAs. |
| `mm_count` | Reference count for the mm_struct itself. |
| `mm_users` | Number of lightweight processes (threads) sharing this mm. |
| `mmap_sem` | Read-write semaphore protecting VMA list/tree modifications. Readers (e.g., page fault handler) take it shared; writers (e.g., `mmap()`, `munmap()`) take it exclusive. |
| `page_table_lock` | Spinlock protecting page table modifications. |
| `start_code` / `end_code` | Text segment boundaries. |
| `start_data` / `end_data` | Initialized data segment boundaries. |
| `start_brk` / `brk` | Heap boundaries (grown by `sys_brk()`). |
| `start_stack` | Stack start address. |
| `arg_start` / `arg_end` | argv region. |
| `env_start` / `env_end` | envp region. |
| `rss` | Resident set size (number of page frames currently mapped). |
| `total_vm` | Total number of pages in the address space. |

## Virtual memory areas: vm_area_struct

A **VMA** (`struct vm_area_struct`) represents a contiguous range of virtual addresses with uniform protection and properties. Examples include the text segment, a shared library mapping, a `mmap()`'d file region, the heap, and the stack.

Key fields:

| Field | Description |
|-------|-------------|
| `vm_start` | Start address (inclusive, page-aligned). |
| `vm_end` | End address (exclusive, page-aligned). |
| `vm_flags` | Protection and behavior flags (see below). |
| `vm_page_prot` | Page-level protection bits (translated from `vm_flags` for hardware PTEs). |
| `vm_ops` | Pointer to `struct vm_operations_struct` with methods: `open`, `close`, `nopage`, `populate`. |
| `vm_file` | For file-backed mappings, points to the `struct file`. |
| `vm_pgoff` | Page offset within `vm_file` where the mapping starts. |
| `vm_mm` | Back-pointer to the owning `mm_struct`. |
| `vm_next` | Next VMA in the sorted linked list. |
| `vm_rb` | Red-black tree node. |
| `anon_vma` | Pointer to `struct anon_vma` for reverse mapping of anonymous pages. See [page-frame-reclaiming](page-frame-reclaiming.md). |
| `vm_private_data` | Opaque pointer for driver use. |

### VMA flags (vm_flags)

Important flags include:

- **`VM_READ`**, **`VM_WRITE`**, **`VM_EXEC`** -- Read, write, execute permissions.
- **`VM_SHARED`** -- Shared mapping (changes visible to other mappers). If clear, the mapping is private (COW semantics).
- **`VM_GROWSDOWN`** -- Stack-like region that grows downward on page fault.
- **`VM_GROWSUP`** -- Grows upward (used on some architectures).
- **`VM_DENYWRITE`** -- Deny write access to the underlying file while mapped for execution.
- **`VM_LOCKED`** -- Pages are locked in memory (`mlock()`), exempt from reclamation.
- **`VM_IO`** -- Memory-mapped I/O region. Not backed by page frames, never dumped in core files.
- **`VM_SEQ_READ`** / **`VM_RAND_READ`** -- Hints for read-ahead behavior (`madvise()`).
- **`VM_HUGETLB`** -- Backed by huge pages (2 MB or 4 MB on x86).
- **`VM_ACCOUNT`** -- The mapping is accounted against committed memory limits.

### VMA organization

VMAs are stored in two overlapping data structures:

1. **Sorted linked list** (`mmap` / `vm_next`) -- Enables efficient sequential traversal (e.g., for `/proc/[pid]/maps`, for `fork()` duplication).
2. **Red-black tree** (`mm_rb` / `vm_rb`) -- Enables O(log n) lookup by virtual address, critical since a process may have hundreds or thousands of VMAs.

Both structures are kept in sync during insertions and deletions.

## VMA operations

### find_vma()

```c
struct vm_area_struct *find_vma(struct mm_struct *mm, unsigned long addr);
```

Returns the first VMA whose `vm_end > addr` (i.e., the VMA that contains `addr` or the nearest VMA above it). Uses `mmap_cache` for the fast path (single-entry cache hit rate is high due to locality) and falls back to the red-black tree for O(log n) lookup.

### do_mmap() and do_munmap()

`do_mmap()` creates a new VMA (or extends an existing one) for `mmap()` system calls:

1. Checks permissions and computes the effective flags.
2. Finds a suitable free region in the address space via `get_unmapped_area()` (top-down search for the default layout, bottom-up for legacy layout).
3. Calls `vma_merge()` to try to merge with adjacent VMAs that have compatible properties.
4. If merging fails, allocates a new `vm_area_struct`, initializes it, and inserts it into both the linked list and red-black tree.
5. For file-backed mappings, calls the file's `mmap` method (e.g., `generic_file_mmap()`) to set up `vm_ops`.
6. Does **not** allocate page frames or set up page table entries -- those are created lazily on page fault.

`do_munmap()` removes a region of the address space:

1. Finds all VMAs overlapping the target range.
2. If a VMA partially overlaps, it is split via `split_vma()` into separate VMAs.
3. Removes the targeted VMAs and frees their page table entries via `unmap_region()` -> `unmap_vmas()`.
4. Flushes the TLB for the unmapped range.

### vma_merge() and split_vma()

**`vma_merge()`** checks whether a new mapping can be merged with the preceding and/or following VMA. Merging is possible when the adjacent VMAs have the same permissions, flags, file (with contiguous offsets), and anonymous VMA structures. Merging reduces the total number of VMAs, saving memory and improving lookup speed.

**`split_vma()`** divides a single VMA into two at a given address. This is needed when `munmap()` or `mprotect()` affects only part of an existing VMA. A new `vm_area_struct` is allocated for the split-off portion.

## Page fault handling

When a process accesses a virtual address that has no valid page table entry, the CPU generates a **page fault** exception (interrupt 14 on x86). The kernel handles this through a multi-stage pipeline.

### Stage 1: do_page_fault()

The x86 page fault handler entry point. It:

1. Reads the faulting address from the **`cr2`** register.
2. Reads the error code from the stack, which encodes: whether the fault was a read or write, whether it occurred in user or kernel mode, and whether a page table entry was present (protection violation) or absent (non-present page).
3. Distinguishes several cases:
   - **Kernel-mode fault in vmalloc range** -- Handled by `vmalloc_fault()` (copies PTE from `init_mm`). See [memory-management](memory-management.md).
   - **Kernel-mode fault with no VMA** -- Bug or bad pointer. Checks the **exception fixup table** (see below) or triggers an oops.
   - **User-mode fault** -- Proceeds to VMA lookup and the generic handler.

### Stage 2: VMA lookup and validation

1. Calls `find_vma()` to locate the VMA containing or nearest to the faulting address.
2. If no VMA is found or `addr < vma->vm_start`:
   - If the VMA has `VM_GROWSDOWN` and the address is within the tolerance window (32 bytes below `vm_start`, accounting for x86 `push`/`pusha` instructions that decrement `esp` before the access), the stack is **expanded** via `expand_stack()`.
   - Otherwise, it is a `SIGSEGV`.
3. Checks whether the access type (read/write/exec) is permitted by `vm_flags`. A write to a non-`VM_WRITE` region is a `SIGSEGV`.

### Stage 3: handle_mm_fault() and handle_pte_fault()

`handle_mm_fault()` walks the page table hierarchy (PGD -> PUD -> PMD -> PTE), allocating intermediate page table pages as needed via `pud_alloc()`, `pmd_alloc()`, `pte_alloc_map()`.

`handle_pte_fault()` examines the PTE and dispatches to one of three sub-handlers:

#### do_no_page() -- Demand paging

Called when the PTE is entirely empty (the page has never been accessed). Two cases:

- **File-backed mapping**: Calls the VMA's `nopage` method (typically `filemap_nopage()`), which looks up the page in the [page-cache](page-cache.md) or reads it from disk. The page is then installed in the PTE.
- **Anonymous mapping**: Calls `do_anonymous_page()`. For **read** faults, the kernel maps the global **`empty_zero_page`** read-only -- a single, shared page frame filled with zeros. This avoids allocating a physical page until the process actually writes. For **write** faults, a new zeroed page frame is allocated and mapped writable.

The zero-page optimization means that a process can `malloc()` large regions and read from them without consuming any physical memory beyond a single shared zero page.

#### do_wp_page() -- Copy-on-Write

Called when a write fault occurs on a present but read-only PTE that is in a `VM_WRITE` VMA. This is the [concept-copy-on-write](../concepts/concept-copy-on-write.md) mechanism:

1. If the page's reference count (`_count`) is 1 (this process is the sole user), simply make the PTE writable. No copy needed.
2. If the page is shared (reference count > 1), allocate a new page frame, copy the contents of the old page to the new one, decrement the old page's reference count, and install the new page with write permission.

COW is the key optimization that makes `fork()` efficient: the parent and child initially share all page frames as read-only. Pages are duplicated only when either process writes to them.

#### do_swap_page() -- Swap-in

Called when the PTE contains a **swap entry** (present bit clear, but non-zero content encoding the swap device and slot). The handler:

1. Looks up the page in the **swap cache** -- if it is there (another process is also swapping in the same page, or writeback has not completed), uses that page.
2. If not in the swap cache, calls `read_swap_cache_async()` to allocate a page and initiate I/O from the swap device.
3. Waits for the page to be read (`lock_page()` / `wait_on_page_locked()`).
4. Installs the page in the process's page table and updates the reverse mapping.
5. Decrements the swap entry's reference count. If no more references exist, the swap slot is freed.

See [page-frame-reclaiming](page-frame-reclaiming.md) for the swap subsystem.

## Stack expansion

User-space stacks grow downward on x86. The stack VMA has the `VM_GROWSDOWN` flag set. When a page fault occurs at an address below the current `vm_start` of the stack VMA, the kernel applies a **tolerance check**: the faulting address must be within 32 bytes below `vm_start` (the 32-byte tolerance accommodates the x86 `pusha` instruction, which decrements the stack pointer by 32 bytes before any access, creating a window where the new `esp` is below the VMA but the access is legitimate) or the address must match the value in `esp` (checking that the fault is a legitimate stack access, not a wild pointer).

If the check passes, `expand_stack()` extends `vm_start` downward (page-aligned) and updates `total_vm`. The kernel also enforces the `RLIMIT_STACK` resource limit -- if the expanded stack would exceed the limit, a `SIGSEGV` is delivered.

## Heap management: sys_brk() and do_brk()

The process heap is the region between `start_brk` (just after BSS) and `brk` (the current program break). The `brk()` system call adjusts the program break:

- **Expanding** the heap: `sys_brk()` calls `do_brk()`, which creates or extends an anonymous, writable VMA covering the new range. Like `mmap()`, no physical pages are allocated -- they are demand-paged on first access.
- **Shrinking** the heap: `sys_brk()` calls `do_munmap()` on the released range, freeing page tables and any resident pages.

`do_brk()` is a simplified version of `do_mmap()` optimized for anonymous, private, writable mappings (the common case for heap expansion).

The C library's `malloc()` implementation uses `brk()` for small allocations and `mmap()` (anonymous) for large allocations (typically >= 128 KB with glibc).

## Address space lifecycle

### Creation: fork() via copy_mm() and dup_mmap()

When a process calls `fork()`, `copy_mm()` creates a new `mm_struct` for the child:

1. Allocates a new `mm_struct` and copies the parent's fields.
2. Allocates a new PGD (top-level page table).
3. Calls `dup_mmap()` to duplicate the parent's VMA list. For each VMA:
   - Allocates a new `vm_area_struct` and copies the parent's fields.
   - For private mappings, marks all page table entries **read-only** in both parent and child (setting up COW).
   - For shared mappings, the page table entries retain their original permissions.
   - Increments reference counts on the underlying page frames.
4. The child gets its own copy of all page tables, but shares the actual page frames with the parent until COW triggers.

With the `CLONE_VM` flag (used by `pthread_create()`), `copy_mm()` simply increments `mm_users` and shares the parent's `mm_struct` -- threads share the entire address space.

### Destruction: exit_mm() and mmput()

When a process exits:

1. `exit_mm()` detaches the process from its `mm_struct` (sets `task_struct->mm = NULL`).
2. Calls `mmput()`, which decrements `mm_users`. If it reaches zero (no more threads using this mm):
   - `exit_mmap()` unmaps all VMAs and frees all page tables and page frames.
   - `mmdrop()` decrements `mm_count`. When that reaches zero, the `mm_struct` itself is freed.

## Kernel-mode fault handling

### Exception fixup tables

When the kernel accesses user-space memory (e.g., `copy_from_user()`), the access might fault if the user provided a bad pointer. The kernel cannot simply deliver a `SIGSEGV` to itself. Instead, it uses **exception fixup tables**: a sorted table of (faulting instruction address, fixup instruction address) pairs.

When `do_page_fault()` handles a kernel-mode fault and cannot resolve it through normal VMA lookup (the address is in user space, which the kernel is accessing on behalf of the user), it searches the fixup table via `search_exception_tables()`. If a matching entry is found, the instruction pointer is redirected to the fixup code, which typically sets an error return value (`-EFAULT`) and continues execution. This mechanism is what makes `copy_from_user()` / `copy_to_user()` safe -- they include the appropriate fixup table entries for every user-space memory access.

### vmalloc_fault()

When a page fault occurs in the vmalloc range (`VMALLOC_START` to `VMALLOC_END`) in kernel mode, `vmalloc_fault()` handles it by copying the PTE from the reference page table (`init_mm.pgd`) to the current process's kernel page tables. This lazy synchronization avoids the cost of updating all processes' page tables when a new vmalloc mapping is created -- only `init_mm` is updated eagerly, and other processes discover the mapping on first access.

## See also

- [memory-management](memory-management.md) -- Physical memory allocation (buddy system, slab, zones)
- [page-frame-reclaiming](page-frame-reclaiming.md) -- Page reclamation, LRU lists, swap subsystem
- [page-cache](page-cache.md) -- File data caching and the address_space abstraction
- [concept-copy-on-write](../concepts/concept-copy-on-write.md) -- COW mechanism central to fork() and demand paging
- [concept-memory-addressing](../concepts/concept-memory-addressing.md) -- x86 paging hardware and address translation
- [virtual-filesystem](virtual-filesystem.md) -- VFS layer providing file-backed VMA operations
- [process-management](process-management.md) -- Process creation, destruction, and the task_struct
