---
type: entity
created: 2026-04-08
updated: 2026-04-09
sources: [understanding-the-linux-kernel, professional-linux-kernel-architecture]
tags: [vfs, filesystem, dentry-cache, pathname-lookup, file-locking]
---

# Virtual Filesystem (VFS)

The Virtual Filesystem (also called the Virtual Filesystem Switch) is an abstraction layer that provides a uniform interface for user-space programs to perform file operations regardless of the underlying filesystem type. Every `read()`, `write()`, `open()`, or `stat()` call passes through the VFS, which dispatches to the correct filesystem-specific implementation via function pointer tables. This design allows Linux to support dozens of filesystem types simultaneously and even stack them (e.g., a loopback mount of an ext3 image on top of an XFS partition).

## Core VFS Objects

The VFS is built around four fundamental object types. Each object pairs an in-memory data structure with an operations structure containing function pointers. This is the kernel's C-language approximation of object-oriented polymorphism: the data structure is the object, and the operations structure is its virtual method table.

### Superblock

The **superblock** object represents a mounted filesystem instance.

**`struct super_block`** holds global filesystem metadata:

- `s_list` -- links into the global list of all superblocks
- `s_dev` -- device identifier
- `s_blocksize`, `s_blocksize_bits` -- block size and its log2
- `s_type` -- pointer to the `file_system_type` structure
- `s_op` -- pointer to superblock operations
- `s_root` -- dentry of the filesystem root directory
- `s_inodes` -- list of all inodes belonging to this superblock
- `s_dirty` -- list of dirty inodes needing writeback
- `s_fs_info` -- pointer to filesystem-specific private data (e.g., `ext2_sb_info`)

**`struct super_operations`** provides methods such as:

- `alloc_inode()` / `destroy_inode()` -- allocate and free inode objects
- `read_inode()` -- populate an inode from disk
- `dirty_inode()` -- mark an inode as modified
- `write_inode()` -- write inode back to disk
- `put_super()` -- release the superblock on unmount
- `write_super()` -- write superblock metadata to disk
- `statfs()` -- return filesystem statistics (used by `df`)
- `remount_fs()` -- handle mount option changes

### Inode

The **inode** object represents a specific file (regular file, directory, symlink, device node, pipe, socket) within a filesystem.

**`struct inode`** contains:

- `i_ino` -- inode number, unique within the filesystem
- `i_mode` -- file type and permission bits
- `i_nlink` -- hard link count
- `i_uid`, `i_gid` -- owner and group
- `i_size` -- file size in bytes
- `i_atime`, `i_mtime`, `i_ctime` -- access, modification, and change times
- `i_blocks` -- number of 512-byte blocks allocated
- `i_op` -- pointer to inode operations
- `i_fop` -- default file operations (copied into `struct file` on open)
- `i_sb` -- pointer to the owning superblock
- `i_mapping` -- pointer to the `address_space` for page cache management
- `i_dentry` -- list of dentries that refer to this inode (hard links)

**`struct inode_operations`** provides methods for namespace operations:

- `create()` -- create a new regular file
- `lookup()` -- resolve a filename component in a directory
- `link()` / `unlink()` -- create and remove hard links
- `symlink()` -- create a symbolic link
- `mkdir()` / `rmdir()` -- create and remove directories
- `mknod()` -- create a device node, pipe, or socket
- `rename()` -- move/rename a file
- `permission()` -- check access rights
- `getattr()` / `setattr()` -- retrieve and set file attributes
- `truncate()` -- change file size

### Dentry

The **dentry** (directory entry) object represents a component in a pathname. Dentries are not stored on disk in most filesystems; they are constructed on the fly during pathname lookup and then cached for performance.

**`struct dentry`** contains:

- `d_name` -- the name component (a `qstr` with hash, length, and character pointer)
- `d_inode` -- pointer to the associated inode (NULL for negative dentries)
- `d_parent` -- pointer to the parent directory's dentry
- `d_subdirs` -- list of child dentries
- `d_child` -- linkage in parent's `d_subdirs` list
- `d_op` -- pointer to dentry operations
- `d_sb` -- pointer to the superblock
- `d_flags` -- flags including `DCACHE_AUTOFS_PENDING`, `DCACHE_NFSEXPORT_FLAGS`
- `d_mounted` -- nonzero if this dentry is a mount point
- `d_count` -- reference count

**`struct dentry_operations`** provides optional methods:

- `d_revalidate()` -- check if a cached dentry is still valid (critical for network filesystems like NFS)
- `d_hash()` -- compute the hash for this dentry name
- `d_compare()` -- compare dentry names (allows case-insensitive filesystems)
- `d_delete()` -- called when the last reference is dropped
- `d_release()` -- called when the dentry is deallocated
- `d_iput()` -- called when the inode association is released

### File

The **file** object represents an open file from a process's perspective. Multiple file objects can refer to the same dentry/inode (e.g., after `dup()` or `fork()`).

**`struct file`** contains:

- `f_dentry` -- pointer to the associated dentry
- `f_vfsmnt` -- pointer to the vfsmount of the containing filesystem
- `f_op` -- pointer to file operations
- `f_pos` -- current file offset (read/write position)
- `f_count` -- reference count
- `f_flags` -- open flags (`O_RDONLY`, `O_NONBLOCK`, etc.)
- `f_mode` -- access mode (`FMODE_READ`, `FMODE_WRITE`)
- `f_mapping` -- pointer to the `address_space` (usually same as `inode->i_mapping`)

**`struct file_operations`** provides the methods most familiar to user-space developers:

- `read()` / `write()` -- transfer data
- `llseek()` -- change the file position
- `readdir()` -- iterate directory entries
- `poll()` -- check for I/O readiness
- `ioctl()` -- device-specific commands
- `mmap()` -- map file into process address space
- `open()` / `release()` -- per-open and per-close hooks
- `flush()` -- called on every `close()` of a file descriptor
- `fsync()` -- synchronize file data to disk
- `lock()` -- file locking operations
- `sendfile()` -- zero-copy data transfer between file descriptors

## Object-Oriented Dispatch

The VFS achieves filesystem independence through function pointer tables. When user space calls `read()`, the syscall handler obtains the `struct file` from the process's file descriptor table, then calls `file->f_op->read()`. This pointer was set when the file was opened, pointing to the filesystem-specific implementation (e.g., `ext3_file_operations.read` which is typically `generic_file_read()`). The same pattern repeats for every operation: the VFS looks up the appropriate operations structure and calls through a function pointer, achieving dynamic dispatch without C++ or runtime type information.

Each filesystem type fills in its own operations structures at mount time (for superblocks), at inode read time (for inodes), at lookup time (for dentries), and at open time (for files). If a filesystem does not implement an optional operation, it sets the function pointer to NULL, and the VFS either provides a default behavior or returns an error.

## Dentry Cache (dcache)

Pathname lookup is one of the most frequent operations in the kernel, and reading directory data from disk for every component would be prohibitively slow. The **dentry cache** (dcache) keeps previously resolved dentries in memory, organized in a hash table for O(1) lookup.

### Hash Table Lookup

The dcache hash table is indexed by a hash of the parent dentry pointer and the name component. The function **`__d_lookup()`** searches the hash chain for a matching dentry. It uses RCU (Read-Copy-Update) to avoid taking locks on the common read path, making concurrent lookups scale well on SMP systems.

### Dentry States

A dentry can be in one of four states:

1. **Free** -- the dentry object has been reclaimed by the slab allocator and does not exist in memory.
2. **Unused** -- the dentry is valid and present in the dcache, but its `d_count` is zero. It remains in the hash table and on an LRU list. If memory pressure rises, unused dentries can be reclaimed by the dcache shrinker.
3. **In-use** -- the dentry is valid and actively referenced (`d_count > 0`). It cannot be reclaimed.
4. **Negative** -- the dentry is not associated with any inode (`d_inode == NULL`). This means a lookup was performed and the name was confirmed not to exist. Negative dentries are valuable because they prevent repeated fruitless disk reads for names that do not exist (e.g., repeated lookups for shared library paths).

### LRU List

Unused and negative dentries are placed on an LRU (Least Recently Used) list. When memory is low, the kernel's `shrink_dcache_memory()` function reclaims the oldest entries. Because each dentry holds a reference to its inode, reclaiming a dentry may also allow the inode to be reclaimed from the inode cache.

## Pathname Resolution

When a process calls `open("/usr/bin/vim", ...)`, the kernel must resolve the pathname string into a dentry. This is one of the most complex operations in the VFS.

### path_lookup()

The entry point is **`path_lookup()`**, which initializes a `nameidata` structure containing:

- `dentry` and `mnt` -- the current resolution point (starts at `/` for absolute paths, or the process's `pwd` for relative paths)
- `last` -- the last component parsed
- `flags` -- lookup flags (`LOOKUP_FOLLOW`, `LOOKUP_DIRECTORY`, `LOOKUP_PARENT`, etc.)

### link_path_walk()

The core loop is **`link_path_walk()`**, which breaks the path into components separated by `/` and resolves each one:

1. **Skip leading slashes** and handle `.` (current directory).
2. **Compute the hash** of the name component.
3. **Handle `..`** by following `d_parent`, taking care at filesystem roots to cross mount boundaries upward.
4. **Call `do_lookup()`** to resolve the component.
5. **Check permissions** (`exec_permission_lite()` or `permission()`) for directory traversal.
6. **Handle mount points** via `follow_mount()`.
7. **Handle symlinks** via `do_follow_link()`.
8. Advance to the next component and repeat.

### do_lookup()

**`do_lookup()`** first attempts a dcache lookup via `__d_lookup()`. On a cache hit, it returns immediately. On a miss, it calls the filesystem's `inode_operations->lookup()` method, which reads the directory from disk, finds the entry, reads the target inode, allocates a new dentry, and inserts it into the dcache.

### Mount Point Traversal

**`follow_mount()`** is called after each component is resolved. If the dentry has `d_mounted` set (nonzero), the function searches the mount hash table for a vfsmount whose mount point matches this dentry. If found, it replaces the current dentry and vfsmount with the root dentry of the mounted filesystem. This is repeated in a loop to handle stacked mounts (multiple filesystems mounted on the same point).

### Symlink Resolution

When a component is a symbolic link, `do_follow_link()` calls the inode's `follow_link()` method, which reads the link target and recursively invokes pathname resolution. To prevent infinite loops from circular symlinks, the kernel enforces two limits:

- **Nested symlink depth**: maximum **5** levels of recursion (symlinks pointing to symlinks). This is tracked via `current->link_count`.
- **Total symlinks followed**: maximum **40** per pathname resolution. This is tracked via `current->total_link_count`.

If either limit is exceeded, the resolution fails with `ELOOP`.

### LOOKUP_PARENT Mode

For operations that create or remove directory entries (`create`, `unlink`, `mkdir`, `rmdir`, `rename`, `link`, `symlink`, `mknod`), the VFS needs to resolve all but the last component and then operate on that last component within its parent directory. The **`LOOKUP_PARENT`** flag instructs `link_path_walk()` to stop one component short: it stores the final component in `nameidata->last` and returns the parent directory's dentry. The caller then uses this information to perform the actual filesystem operation (e.g., calling `dir->i_op->create()` with the last component name).

## Filesystem Registration

Each filesystem type is represented by a **`struct file_system_type`** containing:

- `name` -- the filesystem type name (e.g., "ext3", "nfs", "tmpfs")
- `fs_flags` -- flags such as `FS_REQUIRES_DEV` (needs a block device)
- `get_sb()` -- function to read or create the superblock
- `kill_sb()` -- function to release the superblock
- `next` -- linked list pointer for the global filesystem type list

A filesystem module calls **`register_filesystem()`** to add its `file_system_type` to the global linked list. User space can enumerate registered types via `/proc/filesystems`. When a `mount()` syscall specifies a filesystem type, the VFS searches this list, calls `get_sb()` to obtain a superblock, and creates a new vfsmount.

## Mounting

The **`struct vfsmount`** links a mount point to the mounted filesystem:

- `mnt_sb` -- pointer to the superblock
- `mnt_root` -- dentry of the root of the mounted filesystem
- `mnt_mountpoint` -- dentry of the mount point in the parent filesystem
- `mnt_parent` -- pointer to the parent vfsmount
- `mnt_mounts` -- list of child mounts
- `mnt_flags` -- mount flags (`MNT_NOSUID`, `MNT_NODEV`, `MNT_NOEXEC`, etc.)

A mount hash table, keyed by (parent vfsmount, mount point dentry), enables efficient mount point detection during pathname resolution. Each process has a namespace (`struct namespace`) containing its tree of vfsmounts, enabling per-process mount namespaces.

## File Descriptor Management

Each process has a **`struct files_struct`** (pointed to by `task_struct->files`) containing an `fd` array of `struct file *` pointers (initially inline, dynamically grown), an `open_fds` bitmap, and a `close_on_exec` bitmap. New file descriptors are allocated by scanning `open_fds` for the lowest clear bit.

## sys_open() Call Chain

The `open()` system call follows this path:

1. **`sys_open()`** -- calls `get_unused_fd()`, then `filp_open()`.
2. **`filp_open()`** -- calls `open_namei()` for pathname resolution, then `dentry_open()`.
3. **`open_namei()`** -- resolves path via `path_lookup()`. Handles `O_CREAT`/`O_TRUNC`/`O_EXCL`. For creation: `LOOKUP_PARENT` mode, then `vfs_create()`.
4. **`dentry_open()`** -- allocates `struct file`, sets `f_op` from `inode->i_fop`, calls `f_op->open()`.
5. `fd_install()` installs the file pointer in the fd array.

## File Locking

Linux supports two distinct file locking mechanisms, each with different semantics.

### FL_FLOCK Locks (BSD-style)

`flock()` locks apply to the entire file and are associated with the **file object**. Processes sharing the same file object (via `fork()` or `dup()`) share the lock. Closing any fd referring to the same file object releases the lock. Only whole-file, advisory locking.

### FL_POSIX Locks (POSIX-style)

`fcntl()`-based locks (`F_SETLK`, `F_SETLKW`, `F_GETLK`) are associated with the **(process, inode)** pair. They support byte-range locking, are not inherited across `fork()`, and support both read (shared) and write (exclusive) locks. Notably, closing **any** fd to the file releases all POSIX locks for that process on that file — a surprising POSIX-mandated behavior.

### Lock Storage

All locks are stored as `struct file_lock` objects linked from `inode->i_flock`, recording type (`F_RDLCK`/`F_WRLCK`), flags (`FL_POSIX`/`FL_FLOCK`/`FL_LEASE`), byte range, and owner.

**Advisory locks** are cooperative (unenforced). **Mandatory locking** (set-group-ID + clear group-execute, mount with `mand`) checks locks on every `read()`/`write()`. For blocking POSIX locks, `posix_locks_deadlock()` detects wait-chain cycles, returning `EDEADLK`.

## Special Filesystems

Several pseudo-filesystems have no on-disk representation but use the VFS infrastructure:

- `/proc` -- see [proc-filesystem](proc-filesystem.md)
- `sysfs` -- see [device-driver-model](device-driver-model.md)
- `tmpfs` -- memory-backed filesystem using page cache and swap
- Other: pipefs, sockfs, devpts, bdev, binfmt_misc

## VFS Evolution in Linux 2.6.24

Mauerer (Linux 2.6.24) documents several VFS API changes from the 2.6.11 version described above:

### write_begin() / write_end()

`prepare_write()`/`commit_write()` in `address_space_operations` were replaced by `write_begin()`/`write_end()`. The new `write_end()` takes a `copied` parameter that explicitly handles short copies (e.g., faulting user buffer), fixing subtle data corruption bugs in the old interface.

### struct path

A new `struct path` bundles `dentry` + `vfsmount` together. `nameidata` and `struct file` now use `struct path` instead of separate fields.

### RCU dcache Lookup

`__d_lookup()` uses RCU for lockless hash chain traversal, taking `d_lock` only to verify the match after the RCU-protected read. Significantly improves SMP scalability.

### read_inode() Deprecation

The `read_inode()` superblock operation is deprecated. Filesystems now use `iget5_locked()`/`iget_locked()` to obtain an inode, then populate it before calling `unlock_new_inode()`.

## See also

- [ext2-ext3](ext2-ext3.md)
- [block-layer](block-layer.md)
- [device-driver-model](device-driver-model.md)
- [page-cache](page-cache.md)
- [extended-attributes-acls](extended-attributes-acls.md)
