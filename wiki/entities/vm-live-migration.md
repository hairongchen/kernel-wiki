---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [qemu-kvm-source-code-and-application, mastering-kvm-virtualization]
tags: [migration, live-migration, dirty-pages, vmstate, xbzrle, post-copy]
---

# VM Live Migration

Live migration transfers a running virtual machine from one physical host to another with minimal downtime. In the QEMU/KVM stack, this involves moving both the VM's RAM contents and device state over a network connection while the guest continues to execute. The mechanism relies on iterative dirty-page tracking to converge on a small final transfer, keeping the blackout window short.

## Overview

Migration proceeds in three high-level phases:

1. **Mark all RAM dirty** -- the entire guest memory is considered unsent. A bitmap with one bit per guest page is allocated and initialized to all-ones.
2. **Iterative transfer** -- while the VM keeps running, QEMU repeatedly scans for dirty pages and sends them to the destination. KVM logs which pages the guest modifies between iterations.
3. **Stop-and-copy** -- when the remaining dirty data is small enough to transfer within the configured downtime limit, QEMU pauses the VM, sends the final dirty pages plus all device state, and signals the destination to resume.

Each module that participates in migration is described by a **SaveStateEntry**. These fall into two categories:

- **Live data** (principally RAM): uses `SaveVMHandlers` with callbacks `save_live_setup`, `save_live_iterate`, `save_live_pending`, and `save_live_complete_precopy`. These run while the VM is still executing.
- **Device state**: uses `VMStateDescription` to declaratively serialize register values, queues, and internal structures. This data is sent only after the VM is stopped.

## Migration Thread

A migration begins when the management layer issues a QMP `migrate` command. The call chain is:

```
qmp_migrate()
  -> tcp_start_outgoing_migration()
       -> migrate_fd_connect()
            -> qemu_thread_create("migration", migration_thread, ...)
```

**MigrationState** is the central tracking structure. Key fields include the current status, `to_dst_file` (the `QEMUFile` wrapping the outgoing socket), and `MigrationParameters` (bandwidth cap, downtime limit, compression settings).

The migration thread calls four functions in a loop:

1. `qemu_savevm_state_begin()` -- invokes `save_live_setup` on each live module (e.g., RAM bitmap allocation).
2. `qemu_savevm_state_pending()` -- queries each module for how much data remains unsent, returning `pending_size`.
3. `qemu_savevm_state_iterate()` -- invokes `save_live_iterate` to send a batch of dirty pages.
4. `migration_completion()` -- calls `qemu_savevm_state_complete_precopy()` to finish the transfer.

The loop logic is driven by a threshold:

```
max_size = bandwidth * downtime_limit
```

- While `pending_size >= max_size`: call iterate (keep sending dirty pages while the VM runs).
- When `pending_size < max_size`: the remaining data fits within the downtime budget, so call `migration_completion`.

## RAM Migration

RAM is the largest component and uses the live migration callbacks.

### Setup

`ram_save_setup()` allocates `migration_bitmap_rcu->bmap`, a bitmap with one bit per guest page, all bits set to 1 (every page is initially dirty). It then calls `memory_global_dirty_log_start()`, which propagates down to KVM via `kvm_mem_flags` setting `KVM_MEM_LOG_DIRTY_PAGES` on each memory slot. From this point, KVM hardware-assists dirty page tracking through the MMU.

### Iteration

`ram_save_iterate()` loops calling `ram_find_and_save_block()`, which follows this chain:

```
ram_find_and_save_block()
  -> find_dirty_block()        // scan bitmap for next dirty bit
  -> ram_save_target_page()
       -> ram_save_page()
            -> qemu_put_buffer()  // write page to QEMUFile
```

Each iteration is time-bounded to `MAX_WAIT = 50ms` to avoid holding the global lock too long and starving VCPU threads.

### Pending Calculation

`ram_save_pending()` reports how many bytes remain. When the reported value drops below `max_size`, it triggers `migration_bitmap_sync()` to pull the latest dirty bits from KVM before making the final size determination.

## Dirty Page Synchronization

Two bitmaps track dirty pages at different layers:

- **QEMU migration bitmap**: `migration_bitmap_rcu->bmap` -- consumed by the migration thread to find pages to send.
- **KVM dirty bitmap**: `ram_list.dirty_memory[DIRTY_MEMORY_MIGRATION]` -- populated by KVM's hardware-assisted tracking via the MMU write-protect mechanism.

The synchronization chain transfers bits from the hardware layer up to the migration bitmap:

```
migration_bitmap_sync()
  -> memory_global_dirty_log_sync()
       -> kvm_log_sync()
            -> kvm_physical_sync_dirty_bitmap()
                 -> ioctl(KVM_GET_DIRTY_LOG)
                 -> kvm_get_dirty_pages_log_range()
                      -> cpu_physical_memory_set_dirty_lebitmap()
                           // atomic_or into DirtyMemoryBlocks
```

After the KVM bitmap is merged into the QEMU-level `DirtyMemoryBlocks`, the migration thread copies those bits into its own bitmap:

```
cpu_physical_memory_sync_dirty_bitmap()
  // atomic_xchg to move bits from DirtyMemoryBlocks -> migration_bitmap
```

The `atomic_xchg` ensures each dirty bit is consumed exactly once: the migration thread clears the bit in `DirtyMemoryBlocks` while copying it, so newly dirtied pages are not lost.

## Convergence Control

Three mechanisms help migration converge (the dirty rate must eventually fall below the transfer rate):

### Bandwidth Limiting

`MigrationParameters.max_bandwidth` controls the outgoing data rate. `QEMUFile` enforces this with rate limiting; when the buffer fills faster than the allowed rate, the migration thread calls `g_usleep()` to throttle itself.

### Downtime Threshold

The configured `downtime_limit` determines `max_size`. A larger downtime tolerance means migration can switch to stop-and-copy with more data remaining, finishing sooner but with a longer guest pause.

### VCPU Throttling

When the guest dirties pages faster than they can be sent, `mig_throttle_guest_down()` slows down the VCPUs:

```
mig_throttle_guest_down()
  -> cpu_throttle_set(pct)
       -> creates timer running cpu_throttle_thread on each VCPU
```

`cpu_throttle_thread` puts each VCPU to sleep for a fraction of each time slice. The sleep ratio is calculated as `throttle_ratio = pct / (1 - pct)`, so a 50% throttle setting yields equal sleep and run times. This reduces the dirty rate at the cost of guest performance.

## XBZRLE Compression

**XOR-Based Zero Run Length Encoding** reduces bandwidth for pages that change only partially between iterations.

### Algorithm

1. XOR the current page content with the previously cached version of the same page.
2. The XOR result has zeros wherever the data is unchanged and non-zero bytes where it differs.
3. Run-length encode the zero runs, producing a compact delta.

The wire format is a sequence of:

```
[zero_run_length] [nonzero_length] [nonzero_data] ...
```

### Implementation

A hash-table cache maps guest page addresses to their previously sent content. When a dirty page is found:

- `xbzrle_encode_buffer()` computes the XOR delta and encodes it.
- If the encoded result is smaller than the raw page, send the compressed form.
- Update the cache with the current page content.

On the destination, `xbzrle_decode_buffer()` reverses the process: XOR the received delta with the cached previous content to reconstruct the full page.

XBZRLE is most effective for workloads that modify small portions of each page (e.g., updating counters or timestamps in large data structures). It is less useful when pages change entirely between iterations.

## Post-Copy Migration

Post-copy inverts the default strategy: instead of sending all pages before switching, it starts the VM on the destination immediately and fetches pages on demand.

### Mechanism

Post-copy relies on the `userfaultfd()` system call for userspace page-fault handling:

1. The destination registers guest memory ranges with `UFFDIO_REGISTER`.
2. When the guest accesses a page that has not yet arrived, a page fault is caught by the userfaultfd handler.
3. The handler requests the specific page from the source host.
4. The source sends the page; the destination installs it via `UFFDIO_COPY`.

A background thread on the source proactively pushes remaining pages to reduce the frequency of demand faults.

### Tradeoffs

- **Advantage**: total migration time is shorter because the VM starts running on the destination sooner. Downtime is minimal.
- **Disadvantage**: individual page-fault latencies can cause performance spikes. If the source host crashes after the switchover, pages not yet transferred are lost, leaving the VM in a split-brain state with incomplete memory. There is no safe rollback once post-copy begins.

## VMState Serialization

Device state is serialized declaratively using `VMStateDescription`, avoiding hand-written save/load functions for most devices.

### VMStateDescription

Key fields:

| Field | Purpose |
|-------|---------|
| `name` | Identifies the device in the migration stream |
| `version_id` | Schema version for forward/backward compatibility |
| `pre_save` | Hook called before serialization (prepare transient state) |
| `post_load` | Hook called after deserialization (rebuild derived state) |
| `fields[]` | Array of `VMStateField` describing each piece of state |
| `subsections[]` | Optional conditional sections sent only when needed |

### VMStateField

Each field descriptor contains:

- `name` -- field identifier
- `offset` -- byte offset within the device struct, typically via `offsetof()`
- `size` -- byte size of the field
- `info` -- pointer to a `VMStateInfo` with `put()` and `get()` callbacks for serialization

### VMSTATE Macros

Helper macros generate `VMStateField` entries:

- `VMSTATE_UINT32(field, struct)` -- a single 32-bit integer
- `VMSTATE_ARRAY(field, struct, count, version, info, type)` -- a fixed-size array
- `VMSTATE_STRUCT(field, struct, version, vmsd, type)` -- a nested structure with its own `VMStateDescription`

### Serialization Flow

`vmstate_save_state()` proceeds as:

1. Call `pre_save()` hook if defined (lets the device pack transient data into serializable fields).
2. Iterate over `fields[]`; for each field call `field->info->put()` to write it to the `QEMUFile`.
3. Evaluate and send applicable `subsections[]`.

On the destination, `vmstate_load_state()` reverses the process and calls `post_load()` to let the device rebuild internal caches or re-register with subsystems.

### Example: e1000

The `vmstate_e1000` descriptor includes `pre_save` and `post_load` hooks. `pre_save` packs MAC filter state and compatibility fields; `post_load` rebuilds the receive filter from the deserialized data and re-registers with the network backend.

## Migration Completion

When `pending_size` drops below `max_size`, the migration thread calls `migration_completion()`:

1. **Stop the VM**: `vm_stop_force_state(RUN_STATE_FINISH_MIGRATE)` halts all VCPUs.
2. **Complete live modules**: `qemu_savevm_state_complete_precopy()` invokes `save_live_complete_precopy` on each live module (RAM sends its final dirty pages).
3. **Send device state**: for each registered `SaveStateEntry` with a `VMStateDescription`, serialize the device state and write it to the migration stream.
4. **Mark complete**: set `MIGRATION_STATUS_COMPLETED`.

The destination detects the completion marker, loads all device state, and resumes the VM. From the guest's perspective, there is a brief pause (bounded by `downtime_limit`) during which no instructions execute.

## libvirt Migration Operations

The QEMU-level migration mechanism described above is exposed through libvirt's `virsh migrate` interface, which adds transport, security, and management features.

### Migration Modes

- **Direct**: source QEMU connects directly to destination QEMU for data transfer; libvirtd coordinates setup on both sides.
- **Peer-to-peer (P2P)**: source libvirtd manages the entire process, connecting to destination libvirtd. The admin client only talks to the source. Use `--p2p`.
- **Tunnelled**: migration data tunnelled through the libvirtd connection (encrypted if TLS configured). Use `--tunnelled` (implies `--p2p`).

### Key virsh Flags

| Flag | Purpose |
|------|---------|
| `--live` | Live migration (pre-copy) |
| `--postcopy` | Enable post-copy migration |
| `--persistent` | Define VM on destination |
| `--undefinesource` | Remove VM from source after migration |
| `--copy-storage-all` | Copy all disk storage (no shared storage needed) |
| `--copy-storage-inc` | Copy only changed blocks |
| `--auto-converge` | Throttle guest CPU to reduce dirty rate |
| `--compressed` | Compress memory pages during transfer |

### Tuning

- `virsh migrate-setmaxdowntime <domain> <ms>` -- maximum acceptable pause during final switchover
- `virsh migrate-setspeed <domain> <MiB/s>` -- bandwidth limit for migration traffic
- `virsh domjobinfo <domain>` -- monitor progress (data transferred, remaining, expected downtime)

### CPU Compatibility

- `virsh cpu-compare` -- check if a CPU description is compatible with the host
- `virsh cpu-baseline` -- compute the greatest common CPU model across multiple hosts
- CPU modes: `host-passthrough` (exact host CPU, not migratable), `host-model` (close approximation, migratable), `custom` (explicit model, most portable)

### TLS Security

Migration data can be secured with TLS certificates. Certificates stored at `/etc/pki/libvirt/` (server/client certs + CA). Configure `listen_tls=1` in `libvirtd.conf`. QEMU migration channel uses separate certs at `/etc/pki/libvirt-migrate/`.

## See also

- [qemu-kvm-overview](qemu-kvm-overview.md)
- [kvm-memory-virtualization](kvm-memory-virtualization.md)
- [virtio-framework](virtio-framework.md)
- [libvirt-management](libvirt-management.md)
