---
type: entity
created: 2026-04-08
updated: 2026-04-08
sources: [understanding-the-linux-kernel]
tags: [ext2, ext3, filesystem, journaling, jbd]
---

# Ext2 and Ext3 Filesystems

Ext2 (Second Extended Filesystem) is the classic Linux filesystem, and Ext3 is its journaling successor. Ext3 is essentially Ext2 with a journal bolted on via the JBD (Journaling Block Device) layer. An Ext3 filesystem can be mounted as Ext2 (losing journal protection), and an Ext2 filesystem can be upgraded to Ext3 by adding a journal with `tune2fs -j`. Both filesystems operate through the [virtual-filesystem](virtual-filesystem.md) layer.

## Ext2 Disk Layout

An Ext2 filesystem divides the partition into fixed-size **block groups**. Each block group contains the same structural elements, creating a regular, predictable layout.

### Block Groups

The partition layout is:

```
| Boot Block | Block Group 0 | Block Group 1 | ... | Block Group N |
```

The boot block (1024 bytes at the start of the partition) is reserved for the boot loader and is not managed by the filesystem.

Each block group contains:

| Component | Description |
|-----------|-------------|
| **Superblock copy** | Redundant copy of the filesystem superblock (not in every group -- sparse superblock feature stores copies only in groups 0, 1, and powers of 3, 5, 7) |
| **Group descriptors** | Array describing all block groups: block bitmap location, inode bitmap location, inode table location, free block/inode counts |
| **Data block bitmap** | One block-sized bitmap tracking free/allocated data blocks in this group |
| **Inode bitmap** | One block-sized bitmap tracking free/allocated inodes in this group |
| **Inode table** | Contiguous array of `ext2_inode` structures for this group |
| **Data blocks** | The actual file data blocks |

### Design for Locality

The block group design promotes spatial locality: a file's inode, its directory entry, and its data blocks are ideally all in the same block group, reducing disk seek times. The allocation policy tries to place:

- A new directory in a group with a high free inode count and low directory count (spreading directories across groups).
- A new file's inode in the same group as its parent directory.
- A file's data blocks in the same group as its inode.

This policy means that related data is physically close on disk, which matters enormously for rotational media.

## ext2_inode: Block Addressing

Each inode occupies a fixed 128 bytes on disk (the `ext2_inode` structure). Key fields include the file mode, owner, size, timestamps, link count, and block count. The block addressing scheme uses **15 block pointers**:

- **12 direct pointers** (`i_block[0]` through `i_block[11]`): each points directly to a data block. For a 4 KB block size, this covers files up to 48 KB.
- **1 single-indirect pointer** (`i_block[12]`): points to a block containing an array of direct block pointers. With 4-byte block numbers and 4 KB blocks, this adds 1024 pointers = 4 MB.
- **1 double-indirect pointer** (`i_block[13]`): points to a block of single-indirect pointers. This adds 1024 x 1024 = 1M blocks = 4 GB.
- **1 triple-indirect pointer** (`i_block[14]`): points to a block of double-indirect pointers. This adds 1024^3 blocks = 4 TB (though Ext2 has other limits that cap maximum file size below this).

The function **`ext2_get_block()`** translates a logical block number within a file to a physical block number on disk. It walks the appropriate level of indirection, allocating new blocks as necessary for writes.

## Block Allocation and Preallocation

**`ext2_new_block()`** allocates a new data block. It first checks the block's group descriptor for free blocks, then scans the block bitmap. The algorithm prefers to allocate blocks near the goal block (typically the last block allocated to the file) to maintain contiguity.

**Block preallocation**: when a process writes sequentially, Ext2 preallocates up to **8 contiguous blocks** beyond what is immediately needed. These preallocated blocks are reserved in the bitmap but not yet assigned to the file. If the file continues to grow, the preallocated blocks are consumed without further bitmap scans. If the file is closed or another file needs space, the unused preallocated blocks are released. This strategy dramatically reduces fragmentation for sequentially-written files and amortizes the cost of bitmap operations.

The preallocated blocks are tracked in the in-memory `ext2_inode_info` structure (`i_prealloc_block` and `i_prealloc_count`), not on disk.

## Directory Entries

Ext2 directories are stored as files whose data blocks contain a linked list of **`ext2_dir_entry_2`** structures:

| Field | Size | Description |
|-------|------|-------------|
| `inode` | 4 bytes | Inode number of the referenced file (0 = deleted entry) |
| `rec_len` | 2 bytes | Total size of this directory entry (including padding) |
| `name_len` | 1 byte | Length of the filename |
| `file_type` | 1 byte | File type (regular, directory, symlink, etc.) -- avoids reading the inode just to determine type |
| `name` | up to 255 bytes | The filename, not null-terminated |

Entries are variable-length due to variable filename lengths. When a file is deleted, its entry's `inode` field is set to 0 and its `rec_len` is merged with the previous entry's `rec_len`. This creates free space that can be reused for new entries. Looking up a name in a directory requires a linear scan of these entries, though indexed directories (htree) were added in later Ext2/Ext3 versions to accelerate lookups in large directories.

## Ext3: Journaling with JBD

Ext3 adds crash consistency guarantees via the **Journaling Block Device (JBD)** layer. Without a journal, an unclean shutdown can leave the filesystem in an inconsistent state, requiring a full `fsck` that scans every block -- a process that can take hours on large filesystems. With a journal, recovery takes seconds: the kernel replays committed transactions and discards incomplete ones.

The JBD is a generic layer not specific to Ext3; it can be used by any block-based filesystem.

### Journal Modes

Ext3 supports three journaling modes, selectable at mount time:

| Mode | What is journaled | Behavior | Trade-off |
|------|-------------------|----------|-----------|
| **journal** | Data + metadata | All file data and metadata are written to the journal before being written to their final locations. | Safest but slowest: all data is written twice. |
| **ordered** (default) | Metadata only | Metadata is journaled; data blocks are written to their final locations **before** the metadata transaction commits. | Good balance: guarantees that metadata always points to valid (non-stale) data. No data written twice. |
| **writeback** | Metadata only | Metadata is journaled; data blocks can be written in any order relative to the journal. | Fastest but after a crash, recently-written files may contain stale data from previously-deleted files. |

The **ordered** mode is the default because it prevents the security-sensitive scenario where metadata is updated but data blocks still contain old content from a different file.

### JBD Core Structures

The JBD uses three key structures:

**`handle_t`** (handle): represents a single atomic filesystem operation (e.g., one `write()`, one `unlink()`). A handle groups all the metadata buffer modifications that must be committed together. A filesystem operation calls `journal_start()` to begin a handle and `journal_stop()` to end it.

**`transaction_t`** (transaction): groups multiple handles into a single transaction that will be committed to the journal as a unit. While one transaction is being committed, a new transaction accepts incoming handles. Key fields include:

- `t_state` -- current state in the lifecycle
- `t_buffers` -- list of metadata buffers modified by this transaction
- `t_nr_buffers` -- count of metadata buffers
- `t_handle_count` -- number of active handles

**`journal_t`** (journal): represents the journal itself. Contains:

- `j_sb_buffer` -- the journal superblock buffer
- `j_running_transaction` -- the currently active transaction accepting handles
- `j_committing_transaction` -- the transaction currently being written to disk
- `j_checkpoint_transactions` -- list of committed transactions awaiting checkpointing
- `j_head`, `j_tail` -- head and tail of the circular journal log
- `j_first`, `j_last` -- first and last block numbers of the journal on disk

### Transaction Lifecycle

A transaction passes through these states:

1. **Running** (`T_RUNNING`): the transaction is open and accepting new handles. `journal_start()` adds handles to this transaction.
2. **Locked** (`T_LOCKED`): the transaction is closed to new handles. Existing handles must complete before the commit can proceed. The transaction moves here when a timer expires (typically 5 seconds), the transaction fills up, or an explicit sync is requested.
3. **Flush** (`T_FLUSH`): all dirty metadata buffers belonging to this transaction are being written to the journal area on disk.
4. **Commit** (`T_COMMIT`): the commit record is written to the journal, atomically making the entire transaction durable. Once the commit record is on disk, the transaction is guaranteed to be replayable during recovery.
5. **Finished** (`T_FINISHED`): the commit is complete. The transaction moves to the checkpoint list.

The function **`journal_commit_transaction()`** drives the transition from locked through commit:

1. Lock the transaction and wait for outstanding handles to finish.
2. Write all metadata buffers to the journal (descriptor blocks + data blocks).
3. Wait for all journal I/O to complete.
4. Write the commit block.
5. Wait for the commit block write to complete.
6. Move the transaction to the finished/checkpoint state.

### Checkpointing

Even after a transaction is committed, the journal blocks cannot be reused until the journaled metadata has been written to its **final on-disk location** (the real metadata blocks, not the journal copy). This process is called **checkpointing**.

The checkpointing thread (`kjournald`) periodically writes metadata buffers from committed transactions to their final locations. Once all buffers from a transaction are written to their real locations, the transaction's journal space is reclaimed, and `j_tail` advances. If the journal runs out of space, the commit code forces a synchronous checkpoint to free room.

### Recovery

On mount, if Ext3 detects that the filesystem was not cleanly unmounted, it performs journal recovery:

1. Read the journal superblock to find the journal boundaries.
2. Scan forward from `j_tail` through the journal.
3. For each transaction with a valid commit record, **replay** it: copy the journaled metadata blocks to their final on-disk locations.
4. For any transaction without a commit record (incomplete at crash time), **discard** it: do nothing, as the original on-disk metadata was never updated.
5. Mark the journal as empty.

This replay is idempotent: replaying an already-applied transaction is harmless because the same data is written to the same location. Recovery is fast because only the journal (a small, contiguous area) needs to be read, not the entire filesystem.

## Key Functions Summary

| Function | Purpose |
|----------|---------|
| `ext2_new_block()` | Allocate a new data block from the block bitmap |
| `ext2_get_block()` | Map logical file block to physical disk block |
| `ext2_alloc_inode()` | Allocate a new inode from the inode bitmap |
| `ext2_lookup()` | Look up a filename in an Ext2 directory |
| `journal_start()` | Begin an atomic handle for a filesystem operation |
| `journal_stop()` | End the handle |
| `journal_commit_transaction()` | Commit the current transaction to the journal |
| `journal_recover()` | Replay the journal during mount |

## See also

- [virtual-filesystem](virtual-filesystem.md)
- [block-layer](block-layer.md)
- [page-cache](page-cache.md)
