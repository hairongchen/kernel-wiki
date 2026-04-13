---
type: concept
created: 2026-04-09
updated: 2026-04-09
sources: [advanced-linux-programming]
tags: [assembly, gcc, x86, performance, low-level]
---

# Inline Assembly (GCC)

GCC's inline assembly allows embedding assembly instructions directly within C/C++ code. This is used in the Linux kernel and performance-critical applications for operations that cannot be expressed in C: accessing CPU special registers, atomic operations, SIMD instructions, and precise control over instruction ordering.

## GAS Syntax (AT&T Style)

GCC uses the GNU Assembler (GAS) syntax, which follows AT&T conventions rather than Intel conventions:

| Feature | AT&T (GAS) | Intel |
|---------|-----------|-------|
| Operand order | `mnemonic source, dest` | `mnemonic dest, source` |
| Register names | `%eax`, `%rsp` | `eax`, `rsp` |
| Immediate values | `$42`, `$0xFF` | `42`, `0FFh` |
| Memory reference | `offset(%base, %index, scale)` | `[base + index*scale + offset]` |
| Size suffixes | `movl` (long), `movw` (word), `movb` (byte) | `mov DWORD PTR`, `mov WORD PTR` |

Example: `movl $42, %eax` moves the immediate value 42 into the `eax` register.

## The asm Construct

### Basic asm

```c
asm("assembly instructions");
```

Inserts assembly verbatim. No interaction with C variables — useful only for instructions with no operands or side effects the compiler doesn't need to know about. Rarely used.

### Extended asm

```c
asm volatile (
    "assembly template"
    : output operands      /* optional */
    : input operands       /* optional */
    : clobber list          /* optional */
);
```

Extended asm lets the compiler allocate registers and manage data flow between C and assembly:

- **Output operands**: `"=r"(var)` — the compiler chooses a register, and after execution, stores its value into `var`. The `=` means write-only.
- **Input operands**: `"r"(expr)` — the compiler loads `expr` into a register and substitutes that register name into the template.
- **Clobber list**: registers or memory modified by the assembly that the compiler doesn't know about, e.g., `"memory"`, `"cc"` (condition codes), `"%ecx"`.

Operands are referenced in the template as `%0`, `%1`, etc. (numbered sequentially across outputs then inputs).

## Constraints

Constraints tell the compiler what kinds of locations (registers, memory, immediates) an operand can occupy:

| Constraint | Meaning |
|------------|---------|
| `"r"` | Any general-purpose register |
| `"m"` | Memory operand |
| `"i"` | Integer immediate (compile-time constant) |
| `"a"` | The `%eax` / `%rax` register |
| `"b"` | The `%ebx` / `%rbx` register |
| `"c"` | The `%ecx` / `%rcx` register |
| `"d"` | The `%edx` / `%rdx` register |
| `"S"` | The `%esi` / `%rsi` register |
| `"D"` | The `%edi` / `%rdi` register |
| `"0"`, `"1"`, ... | Same location as operand N (input tied to output) |

### Modifier prefixes

| Prefix | Meaning |
|--------|---------|
| `=` | Output-only (write) |
| `+` | Read-write (both input and output) |
| `&` | Early clobber — register is written before all inputs are consumed |

## Volatile

`asm volatile(...)` tells the compiler:
- Do not remove this asm block even if its outputs appear unused.
- Do not reorder this asm block relative to other volatile operations.

Without `volatile`, the compiler may optimize away the assembly or move it. Always use `volatile` for asm blocks with side effects (I/O port access, memory barriers, timing instructions).

## Common Use Cases

### Read Time-Stamp Counter (rdtsc)

```c
unsigned long long rdtsc(void) {
    unsigned int lo, hi;
    asm volatile ("rdtsc" : "=a"(lo), "=d"(hi));
    return ((unsigned long long)hi << 32) | lo;
}
```

### CPUID

```c
void cpuid(unsigned int leaf, unsigned int *eax, unsigned int *ebx,
           unsigned int *ecx, unsigned int *edx) {
    asm volatile ("cpuid"
        : "=a"(*eax), "=b"(*ebx), "=c"(*ecx), "=d"(*edx)
        : "a"(leaf));
}
```

### Memory Barrier

```c
asm volatile ("mfence" ::: "memory");
```

The `"memory"` clobber tells the compiler that the asm may read or write arbitrary memory, preventing the compiler from reordering memory accesses across this point.

## Relationship to the Kernel

The Linux kernel uses inline assembly extensively for [concept-kernel-synchronization](concept-kernel-synchronization.md) primitives (atomic operations, spinlock implementation), [concept-memory-addressing](concept-memory-addressing.md) (page table manipulation, TLB flushes), [interrupt-handling](../entities/interrupt-handling.md) (interrupt enable/disable), and [system-calls](../entities/system-calls.md) (`sysenter`/`sysexit` entry points). The kernel's use of inline assembly is much more pervasive than typical user-space code because it must interact directly with hardware.

## See also

- [system-calls](../entities/system-calls.md)
- [concept-kernel-synchronization](concept-kernel-synchronization.md)
- [concept-memory-addressing](concept-memory-addressing.md)
