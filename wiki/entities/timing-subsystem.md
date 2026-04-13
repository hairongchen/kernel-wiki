---
type: entity
created: 2026-04-08
updated: 2026-04-08
sources: [understanding-the-linux-kernel]
tags: [timing, jiffies, timers, clock, hpet]
---

# Timing Subsystem

The kernel's timing subsystem provides the heartbeat that drives process scheduling, timer expiration, time-of-day maintenance, and delay calibration. It relies on a hierarchy of hardware timers with varying resolution and purpose, abstracted behind a uniform software interface.

## Hardware Timers

Linux 2.6.11 on x86 supports several hardware timer sources, each with different characteristics:

| Timer | Type | Resolution | Purpose |
|-------|------|------------|---------|
| **Real Time Clock (RTC)** | Battery-backed CMOS chip (MC146818 or compatible) | 1 second (base); up to 8192 Hz periodic | Provides persistent wall-clock time across reboots. The kernel reads it once at boot to initialize `xtime`, then relies on other timers |
| **Time Stamp Counter (TSC)** | Per-CPU 64-bit register, incremented every CPU clock cycle | Sub-nanosecond (at GHz clock rates) | Highest-resolution timer. Read via `rdtsc` instruction. Used for fine-grained interval measurement. Problematic on SMP with variable clock speeds |
| **Programmable Interval Timer (PIT / 8254)** | External chip, 1.193182 MHz oscillator | ~838 ns | The traditional PC timer. Programmed to generate IRQ 0 at the `HZ` rate (1000 Hz in 2.6.11). Used as the system tick source on uniprocessor systems |
| **Local APIC Timer** | Per-CPU, in the local APIC | Based on bus clock | One-shot or periodic timer local to each CPU. Used on SMP systems so each CPU gets its own timer interrupt. Calibrated against the PIT at boot |
| **High Precision Event Timer (HPET)** | Chipset-level timer block (spec by Intel/Microsoft) | ~10 ns (at least 10 MHz) | Modern replacement for PIT and RTC periodic timer. Provides multiple comparators. Preferred when available |
| **ACPI Power Management Timer (ACPI PMT)** | Chipset, 3.579545 MHz | ~279 ns | 24-bit or 32-bit free-running counter. Not affected by power management or CPU frequency changes. Good for measuring intervals on laptops with variable CPU speed |

### cur_timer Abstraction

The kernel selects the best available timer source at boot time and stores a pointer in `cur_timer`. This abstraction provides `mark_offset()` (called at each tick to note the precise time) and `get_offset()` (returns the sub-tick elapsed time since the last tick). The selection priority is typically: HPET > ACPI PMT > TSC > PIT.

## The Tick: jiffies and HZ

The global variable `jiffies` is a counter incremented once per timer tick. The tick rate is set by the `HZ` compile-time constant:

- **Linux 2.6.11 on x86: HZ = 1000** (1 ms per tick)
- Older kernels used HZ = 100 (10 ms per tick)

The `jiffies` variable is declared as `unsigned long` (32-bit on x86), which would overflow after approximately 49.7 days at HZ=1000. The kernel also maintains `jiffies_64`, a full 64-bit counter that will not overflow for hundreds of millions of years. On 32-bit architectures, `jiffies` is aliased to the lower 32 bits of `jiffies_64` via linker trickery.

The kernel provides overflow-safe comparison macros:

```c
time_after(a, b)        /* true if a is after b (handles wraparound) */
time_before(a, b)       /* true if a is before b */
time_after_eq(a, b)     /* a >= b */
time_before_eq(a, b)    /* a <= b */
```

### Converting Between Jiffies and Human Time

```c
msecs_to_jiffies(ms)    /* milliseconds to jiffies */
jiffies_to_msecs(j)     /* jiffies to milliseconds */
```

At HZ=1000, one jiffy equals exactly 1 ms. At other HZ values, rounding occurs.

## Timer Interrupt Processing

The PIT (or HPET, or local APIC timer on SMP) generates IRQ 0 at HZ frequency. The interrupt handler flow is:

```
timer_interrupt()                    /* arch/i386 handler for IRQ 0 */
  -> cur_timer->mark_offset()       /* record precise time of tick */
  -> do_timer_interrupt()
    -> do_timer()                   /* global, once per tick */
      -> jiffies_64++
      -> update_times()
        -> update_wall_time()       /* advance xtime by one tick, apply NTP adj */
        -> calc_load()              /* update loadavg (1, 5, 15 min) */
    -> update_process_times()       /* per-CPU */
      -> account_process_tick()     /* charge tick to user or system time */
      -> run_local_timers()         /* raise TIMER_SOFTIRQ */
      -> scheduler_tick()           /* update current process time slice */
```

On SMP systems, `do_timer()` runs only on one CPU (the "tick CPU"), while `update_process_times()` runs on every CPU via the local APIC timer interrupt.

## Wall Clock: xtime

The `xtime` variable (a `struct timespec` with seconds and nanoseconds) holds the current wall-clock time (UTC since the Unix epoch). It is the kernel's master clock for `gettimeofday()` and related calls.

### Seqlock Protection

`xtime` is protected by `xtime_lock`, a **seqlock** (see [concept-kernel-synchronization](../concepts/concept-kernel-synchronization.md)). The timer interrupt (writer) acquires the write side; readers use the optimistic read side:

```c
unsigned long seq;
do {
    seq = read_seqbegin(&xtime_lock);
    /* read xtime */
} while (read_seqretry(&xtime_lock, seq));
```

This is ideal because `xtime` is written infrequently (once per tick = every 1 ms) but read very frequently (`gettimeofday()` is one of the most called system calls).

### wall_to_monotonic

The `wall_to_monotonic` variable stores the offset between wall-clock time and monotonic time. Wall-clock time can be adjusted (by `settimeofday()` or NTP), but monotonic time (accessible via `clock_gettime(CLOCK_MONOTONIC, ...)`) only moves forward. The kernel computes monotonic time as `xtime + wall_to_monotonic`.

## Dynamic Timer Wheel

The kernel supports an arbitrary number of **dynamic timers** (also called kernel timers or timer-wheel timers). These are one-shot callbacks that fire after a specified number of jiffies. They are the kernel's primary mechanism for implementing timeouts, retransmission timers, polling intervals, and delayed work.

### Data Structure: tvec_base_t

Each CPU has a `tvec_base_t` containing five groups of timer lists organized as a **hierarchical timer wheel**:

| Group | Lists | Covers jiffies | Range |
|-------|-------|-----------------|-------|
| `tv1` | 256 (`TVR_SIZE`) | `expires` bits 0-7 | Now to +255 ticks (~256 ms at HZ=1000) |
| `tv2` | 64 (`TVN_SIZE`) | `expires` bits 8-13 | +256 to +16383 ticks (~16 s) |
| `tv3` | 64 | `expires` bits 14-19 | +16384 to +1048575 ticks (~17 min) |
| `tv4` | 64 | `expires` bits 20-25 | +1048576 to +67108863 ticks (~18 hr) |
| `tv5` | 64 | `expires` bits 26-31 | +67108864 to +4294967295 ticks (~49.7 days) |

### Operations

**Insertion** — `add_timer(timer)`: Compute which group and which list within that group the timer belongs to based on `timer->expires - base->timer_jiffies`. Insert into the appropriate doubly-linked list. This is **O(1)**.

**Cascading**: On each tick, the kernel processes all timers in `tv1.vec[index]` where `index = base->timer_jiffies & 0xFF`. When the tv1 index wraps around to 0 (every 256 ticks), the kernel **cascades**: it moves all timers from the appropriate tv2 list down into tv1, redistributing them among tv1's 256 slots. Similarly, tv3 cascades into tv2 every 256*64 ticks, and so on.

**Execution**: `run_timer_softirq()` (triggered by `TIMER_SOFTIRQ`) processes expired timers by walking the current tv1 slot and calling each timer's callback function.

### Timer API

```c
struct timer_list {
    struct list_head entry;
    unsigned long expires;        /* absolute jiffies value */
    void (*function)(unsigned long);
    unsigned long data;
    struct tvec_base_t *base;
};

void init_timer(struct timer_list *timer);
void add_timer(struct timer_list *timer);
int  mod_timer(struct timer_list *timer, unsigned long expires);  /* atomic modify+add */
int  del_timer(struct timer_list *timer);
int  del_timer_sync(struct timer_list *timer);  /* waits for handler to finish on SMP */
```

`mod_timer()` is the preferred way to change a timer's expiry — it atomically removes the timer from its current position and re-inserts it, and is safe to call even if the timer is not yet active.

## Delay Functions

For short, busy-wait delays (typically in driver initialization or hardware programming):

| Function | Precision | Implementation |
|----------|-----------|----------------|
| `udelay(usecs)` | Microseconds | Calibrated busy-loop using `loops_per_jiffy` |
| `ndelay(nsecs)` | Nanoseconds | Same, scaled for nanoseconds (may round to microseconds on slower CPUs) |
| `mdelay(msecs)` | Milliseconds | Calls `udelay()` in a loop |

### BogoMIPS Calibration

At boot, the kernel calibrates `loops_per_jiffy` by running a tight delay loop and measuring how many iterations fit in one jiffy. This value is reported as **BogoMIPS** (Bogus MIPS) in `/proc/cpuinfo` and `dmesg`. The formula is `BogoMIPS = loops_per_jiffy * HZ / 500000`. BogoMIPS is not a meaningful benchmark — it is solely used to calibrate `udelay()`.

## Sleeping Delays

For longer delays where busy-waiting would waste CPU:

```c
schedule_timeout(timeout);
```

This puts the current process to sleep for `timeout` jiffies. It works by creating a dynamic timer that wakes the process, then calling `schedule()`. The process is removed from the runqueue and re-inserted when the timer fires. The actual delay may be slightly longer than requested (by up to one tick) due to timer granularity.

Variants: `msleep(ms)` and `ssleep(s)` are convenience wrappers. `msleep_interruptible()` allows the sleep to be interrupted by signals.

## POSIX Clocks and Timers

The kernel implements the POSIX clock/timer interface:

- **Clocks**: `CLOCK_REALTIME` (wall clock = `xtime`), `CLOCK_MONOTONIC` (monotonic = `xtime + wall_to_monotonic`). Accessed via `clock_gettime()` / `clock_settime()`.
- **Timers**: Per-process timers created with `timer_create()`, armed with `timer_settime()`. Can deliver signals (`SIGALRM`, or a user-chosen signal via `sigevent`) or be queried with `timer_getoverrun()` for missed expirations. Internally backed by the dynamic timer wheel.

### Process Time Accounting

Each process tracks CPU time in `task_struct`:
- `utime` — ticks spent in user mode
- `stime` — ticks spent in kernel mode

On each timer tick, `account_process_tick()` charges the tick to either `utime` or `stime` depending on whether the interrupted context was user or kernel mode. This is an approximation — a process that enters the kernel briefly near a tick boundary may have its time misattributed.

## NTP: Network Time Protocol

The kernel supports NTP-based clock discipline via the `adjtimex()` system call. NTP allows gradual adjustment of `xtime` by **slewing** the clock (adding or subtracting a small offset each tick) rather than stepping it. The key fields in the `timex` structure include:

- `offset` — time offset to correct (nanoseconds)
- `freq` — frequency offset (scaled PPM)
- `tick` — microseconds per tick (nominal = 10000 at HZ=100, 1000 at HZ=1000)

`update_wall_time()` applies the NTP adjustment on each tick, producing a smooth, drift-corrected clock.

## See also

- [interrupt-handling](interrupt-handling.md)
- [concept-kernel-synchronization](../concepts/concept-kernel-synchronization.md)
- [process-management](process-management.md)
