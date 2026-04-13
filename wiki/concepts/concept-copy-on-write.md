---
type: concept
created: 2026-04-08
updated: 2026-04-08
sources: [understanding-the-linux-kernel]
tags: [copy-on-write, fork, memory-management, demand-paging]
---

# Copy-on-Write (COW)

Copy-on-Write is a memory optimization technique that defers the duplication of memory pages until one of the sharing processes actually modifies a page. It is central to the efficiency of `fork()` in Linux, turning what would be an O(n) operation (copying the entire address space) into an O(1) operation (duplicating only page table entries and metadata).

## Mechanism During fork()

When a process calls `fork()`, the kernel invokes `do_fork()` -> `copy_process()` -> `copy_mm()` -> `dup_mmap()` to create the child's address space. Rather than copying every page of the parent's memory:

1. **dup_mmap()** iterates over each VMA (virtual memory area) in the parent's memory map and creates a corresponding VMA in the child.

2. For each **private, writable** page: both the parent's and child's page table entries (PTEs) are set to **read-only**. The actual page frame is shared — both PTEs point to the same physical page. The page frame's reference count (`_count` in `struct page`) is incremented.

3. For **shared** mappings (e.g., `MAP_SHARED`): both PTEs continue to point to the same page with the original permissions. No COW is applied — writes from either process are visible to the other.

4. For **read-only** pages (e.g., `.text` segments): the PTEs already lack write permission, so no change is needed. These pages are naturally shared.

The result is that immediately after `fork()`, parent and child share all their physical memory pages. The only newly allocated memory is for the child's page tables, `mm_struct`, and VMAs — a very small amount compared to the full address space.

## Write Fault: do_wp_page()

When either the parent or child attempts to write to a COW page, the CPU generates a **Page Fault** (exception #14) because the PTE is marked read-only. The page fault handler (`do_page_fault()`) determines that this is a write to a legitimately writable VMA whose PTE was made read-only for COW, and dispatches to `do_wp_page()`.

`do_wp_page()` implements the actual copy:

1. **Check the reference count** of the page frame (`page_count(old_page)`):

   - If `_count == 1` (the page has only one mapping — the other process has already faulted or exited): **no copy is needed**. The kernel simply marks the PTE writable and returns. This is an important optimization — it avoids unnecessary page copies.

   - If `_count > 1` (the page is still shared):
     a. **Allocate a new page frame** via `alloc_page()`.
     b. **Copy the contents** of the old page to the new page (`copy_page()`, which is typically an optimized `memcpy` of 4096 bytes, using `rep movsl` or MMX/SSE instructions on x86).
     c. **Install the new PTE**: update the faulting process's page table entry to point to the new page frame with full write permissions.
     d. **Decrement** the reference count on the old page frame. If this was the last COW reference, the other process's PTE is still read-only but now has sole ownership — the next write fault from that process will hit the `_count == 1` fast path.

2. **Flush the TLB** entry for the faulted virtual address so the CPU picks up the new PTE.

3. **Return to the faulting instruction**, which re-executes successfully.

## Anonymous Zero-Page Optimization

COW interacts with another demand-paging optimization: the **anonymous zero-page**.

When a process reads from an anonymous page (e.g., heap, BSS) that has never been written:

1. The PTE is initially **not present** (no page frame allocated).
2. The page fault handler detects a read fault on an anonymous VMA.
3. Instead of allocating a new zeroed page, the kernel maps the global `empty_zero_page` (a single page frame permanently filled with zeros) as **read-only** into the faulting PTE.
4. All such read-only anonymous zero-page mappings point to the same physical page, saving memory.

When the process eventually writes to this address:

1. A write fault occurs (the PTE is read-only).
2. `do_wp_page()` sees that the page is the zero-page (shared), allocates a new page, copies the zeros, and installs a writable PTE pointing to the new page.

This means a process can `malloc()` and read large regions without consuming any physical memory — pages are allocated only for locations that are actually written. This is the essence of demand paging.

## The fork() + exec() Pattern

COW makes the common `fork()` + `exec()` pattern extremely efficient:

1. The parent calls `fork()`. Due to COW, this is fast — only metadata is duplicated.
2. The child calls `exec()`. `exec()` calls `flush_old_exec()`, which tears down the child's entire address space (all those shared, read-only PTEs are simply discarded) and replaces it with the new program's segments.
3. **No page was ever copied.** The COW pages set up in step 1 are never faulted on by the child — they are released before any write occurs.

Without COW, `fork()` would have to copy every page, only for `exec()` to immediately discard them all. COW eliminates this waste.

### vfork() as a Historical Alternative

Before COW was implemented efficiently, `vfork()` was introduced as a performance hack: the child borrows the parent's address space directly (parent blocks until child calls `exec()` or `_exit()`). With modern COW, `vfork()` provides only marginal benefit over `fork()` (it saves the cost of duplicating page tables and VMAs). Linux still supports it via `clone()` with `CLONE_VFORK | CLONE_VM`.

## Interaction with Memory-Mapped Files

Private file mappings (`MAP_PRIVATE`) also use COW:

- The file's pages are initially mapped read-only from the page cache.
- On write, a COW fault allocates a private copy (an **anonymous page**) for the writing process.
- The private copy is not written back to the file — it exists only in the process's address space.

This is how processes get their own modifiable copies of shared library data sections (`.data`).

## Performance Characteristics

| Operation | Cost |
|-----------|------|
| `fork()` with COW | O(VMAs) — proportional to the number of memory regions, not pages. Typically hundreds of VMAs vs. millions of pages |
| First write to a COW page | One page allocation + one 4 KB copy + TLB flush |
| Read of a COW page | Zero cost (the page is mapped, just read-only) |
| `exec()` after `fork()` | Discards all shared mappings — effectively free |

## See also

- [process-management](../entities/process-management.md)
- [concept-memory-addressing](concept-memory-addressing.md)
- [program-execution](../entities/program-execution.md)
