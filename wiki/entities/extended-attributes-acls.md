---
type: entity
created: 2026-04-09
updated: 2026-04-09
sources: [professional-linux-kernel-architecture]
tags: [filesystem, xattr, acl, permissions, security]
---

# Extended Attributes and POSIX Access Control Lists

## Overview

Extended attributes (xattrs) provide a generic mechanism for attaching arbitrary name-value pairs to filesystem objects beyond the standard inode metadata (owner, permissions, timestamps, size). Each attribute consists of a null-terminated string name and an opaque binary value of variable length.

POSIX Access Control Lists (ACLs) are built on top of the xattr infrastructure, using the `system` namespace to store fine-grained per-user and per-group permission entries. While the traditional Unix permission model only distinguishes between owner, group, and other, ACLs allow administrators to grant or deny access to specific additional users and groups on a per-file basis.

Both mechanisms were mature features in the Linux 2.6.24 kernel, with support across the major on-disk filesystems (ext2, ext3, ext4, XFS, JFS, ReiserFS).

## Extended Attributes

### Namespaces

Extended attribute names are partitioned into four namespaces, each with distinct access semantics:

| Namespace   | Prefix       | Purpose                                                       | Access Control                          |
|-------------|--------------|---------------------------------------------------------------|-----------------------------------------|
| `user`      | `user.`      | Arbitrary data for user-space applications                    | Read/write governed by file permissions |
| `system`    | `system.`    | Kernel subsystems (ACLs, capabilities)                        | Kernel-managed; policy-specific         |
| `security`  | `security.`  | Security modules such as [SELinux](../concepts/concept-linux-security.md) | Governed by the active LSM              |
| `trusted`   | `trusted.`   | Trusted processes only                                        | Requires `CAP_SYS_ADMIN`               |

The namespace prefix is part of the attribute name. For example, the access ACL is stored under the name `system.posix_acl_access`.

### Kernel Data Structures

Each namespace registers a handler via `struct xattr_handler`, which provides three operations:

- **get()** -- retrieve the value of an attribute in this namespace
- **set()** -- store or update an attribute value (also used for deletion when called with a NULL value)
- **list()** -- enumerate attribute names in this namespace for a given inode

A filesystem registers its supported handlers through the `s_xattr` array on the superblock (`struct super_block`). The VFS iterates this array to find the handler matching the namespace prefix of a requested attribute.

### User-Space System Calls

Four system calls expose xattrs to user space (plus `l` and `f` variants for symlinks and file descriptors):

- `getxattr(path, name, value, size)` -- read a single attribute
- `setxattr(path, name, value, size, flags)` -- create or replace an attribute
- `listxattr(path, list, size)` -- list all attribute names on a file
- `removexattr(path, name)` -- delete an attribute

### VFS Dispatch Path

When user space calls `getxattr()`, the VFS resolves the path to a dentry and calls `inode_operations->getxattr()`. The filesystem implementation looks up the matching `xattr_handler` from `s_xattr` based on the name prefix and delegates to the handler's `get()` method. The `setxattr()` path works analogously through `inode_operations->setxattr()`.

## On-Disk Storage (Ext2/Ext3/Ext4)

### Ext2 and Ext3

In ext2 and ext3, extended attributes for a given inode are stored in a single dedicated disk block. The block number is recorded in `inode->i_file_acl`. The block has the following layout:

1. **Header** -- contains a magic number (`0xEA020000`) and a reference count
2. **Entry array** -- grows forward from after the header. Each entry contains:
   - A name index (identifies the namespace, avoiding redundant prefix storage)
   - The attribute name (the portion after the namespace prefix)
   - The offset and size of the value within the block
3. **Values** -- stored from the end of the block backward, filling toward the entry array

Multiple inodes can share the same xattr block when they carry identical attribute sets. The reference count in the header tracks how many inodes point to the block. The `mb_cache` subsystem maintains a hash table of xattr blocks to enable this sharing: before allocating a new block, the kernel checks whether an existing block with identical content can be reused.

### Ext4 Inline Storage

Ext4 introduces an optimization for small xattrs. When the on-disk inode size exceeds the original 128 bytes (commonly 256 bytes on modern ext4 filesystems), the space between the end of the fixed inode fields and the end of the on-disk inode structure is available for inline xattr storage. Small attributes are placed directly in this space, avoiding the need to allocate a separate disk block. If the inline space is exhausted, ext4 falls back to the traditional external block approach.

## POSIX Access Control Lists

### Data Structures

The kernel represents an ACL as a `struct posix_acl`, which contains:

- `a_count` -- the number of entries in the ACL
- `a_entries[]` -- a variable-length array of `struct posix_acl_entry`

Each `struct posix_acl_entry` has three fields:

- `e_tag` -- the type of entry (see below)
- `e_id` -- the UID or GID this entry applies to (meaningful only for `ACL_USER` and `ACL_GROUP`)
- `e_perm` -- a bitmask of read, write, and execute permissions

### Entry Types

| Tag              | Meaning                                              |
|------------------|------------------------------------------------------|
| `ACL_USER_OBJ`   | Permissions for the file owner                       |
| `ACL_USER`       | Permissions for a specific named user (by UID)       |
| `ACL_GROUP_OBJ`  | Permissions for the file's owning group              |
| `ACL_GROUP`      | Permissions for a specific named group (by GID)      |
| `ACL_MASK`       | Upper bound on permissions for user/group entries     |
| `ACL_OTHER`      | Permissions for everyone else                        |

A minimal ACL contains exactly three entries: `ACL_USER_OBJ`, `ACL_GROUP_OBJ`, and `ACL_OTHER`, which correspond directly to the traditional `rwxrwxrwx` permission bits. An extended ACL adds `ACL_USER` and/or `ACL_GROUP` entries and must also include an `ACL_MASK` entry.

### Permission Algorithm

The function `posix_acl_permission()` evaluates ACL entries in a defined order to determine whether a process is granted access:

1. **Owner check** -- if the process's effective UID matches the file owner, the `ACL_USER_OBJ` entry is used directly. No masking is applied.
2. **Named user check** -- if the effective UID matches an `ACL_USER` entry, that entry's permissions are intersected (bitwise AND) with the `ACL_MASK`. Access is granted only if the requested permission survives the mask.
3. **Group check** -- the kernel collects the highest (most permissive) grant from all matching group entries:
   - The `ACL_GROUP_OBJ` entry is checked if the process belongs to the file's owning group.
   - Each `ACL_GROUP` entry is checked if the process belongs to the named group.
   - The union of matched group permissions is then intersected with `ACL_MASK`.
4. **Other** -- if no earlier rule matched, the `ACL_OTHER` entry is used. No masking is applied.

The `ACL_MASK` entry acts as an upper bound on the effective permissions of all `ACL_USER`, `ACL_GROUP_OBJ`, and `ACL_GROUP` entries. It does not restrict `ACL_USER_OBJ` or `ACL_OTHER`.

### ACL Storage as Extended Attributes

ACLs are serialized into a portable binary format and stored as xattrs in the `system` namespace:

- **Access ACL**: stored under the name `system.posix_acl_access`. Governs permission checks on the file.
- **Default ACL**: stored under the name `system.posix_acl_default`. Only meaningful on directories. When a new file or subdirectory is created inside a directory that carries a default ACL, the new object inherits the default ACL as its access ACL (and subdirectories also inherit it as their own default ACL).

The on-disk format consists of a version header followed by a sequence of entries, each encoding the tag, permission bits, and (where applicable) the UID or GID.

### Interaction with chmod()

When `chmod()` is called on a file that carries an extended ACL, the kernel must synchronize the ACL with the new permission bits:

- The owner bits (`rwx` for user) are written to the `ACL_USER_OBJ` entry.
- The group bits are written to the `ACL_MASK` entry (not `ACL_GROUP_OBJ`), because `ACL_MASK` is what `ls -l` reports as the group permission.
- The other bits are written to the `ACL_OTHER` entry.

Conversely, when an ACL is set on a file, the traditional permission bits visible via `stat()` are updated to reflect `ACL_USER_OBJ`, `ACL_MASK`, and `ACL_OTHER`.

The `ls -l` command appends a `+` character after the permission string (e.g., `rwxr-x---+`) to indicate that the file carries an extended ACL beyond the minimal three entries.

## VFS Integration

The VFS function `generic_permission()` is the central access-check routine. When an inode has ACLs (indicated by a non-null `i_acl` pointer), `generic_permission()` calls `posix_acl_permission()` instead of performing the standard three-category permission check.

Filesystems that support ACLs provide `getxattr` and `setxattr` methods in their `inode_operations`. These methods handle:

- Deserializing the on-disk xattr format into `struct posix_acl`
- Caching the parsed ACL in the inode's `i_acl` and `i_default_acl` fields to avoid repeated disk reads
- Invalidating the cache when the ACL is modified

The caching is important for performance because permission checks occur on every path component during pathname resolution, making repeated disk I/O for ACL data prohibitively expensive.

## See also

- [virtual-filesystem](virtual-filesystem.md)
- [ext2-ext3](ext2-ext3.md)
- [concept-linux-security](../concepts/concept-linux-security.md)
