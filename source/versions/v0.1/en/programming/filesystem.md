# Filesystem Object Types

MuduDB can store file content in the database through **filesystem (fs) object types**. An fs object type is registered by DDL, and the database organizes and manages all storage of its objects. Each object is identified by an OID, its content is stored as immutable *generation* files, and its metadata lives in the `_fs_object` system table, which shares the same concurrency control (MVCC) and WAL mechanisms as ordinary tables. Procedures access objects through a set of POSIX-style syscalls; guests only ever see OIDs and file descriptors, never host paths.

## Registering filesystem types

```sql
CREATE TYPE FILESYSTEM FILE photo_fs;
CREATE TYPE FILESYSTEM DIRECTORY asset_fs;
DROP TYPE photo_fs;
```

- `FILE` is a single-content object; `DIRECTORY` is an object holding an immutable tree of named entries.
- `CREATE TYPE FILESYSTEM` / `DROP TYPE` require an admin session; other sessions fail with `PermissionDenied`.
- A duplicate type name fails with `AlreadyExists`.
- `DROP TYPE` is refused with `InvalidState` while a table column still references the type.
- Using an unknown fs type name as a column type in `CREATE TABLE` fails with `EntityNotFound`.

## FS columns

A table binds an fs object type by using the type name as a column type:

```sql
CREATE TABLE product (
    id BIGINT PRIMARY KEY,
    photo photo_fs
);
```

- The column physically stores the object OID (a `U128` value); the catalog records the column-to-fs-type binding.
- On `INSERT` / `UPDATE`, when the FS column is omitted or set to `NULL`, the kernel creates a new object for the row and assigns its OID automatically (the OID carries a `0xF5` tag in its top byte). Supplying an explicit FS column value is rejected with `InvalidArgument` — objects are created only through FS-column DML.
- `SELECT` of the FS column returns the object OID; visibility follows the same MVCC rules as ordinary columns, including rows not yet committed by the current transaction.
- Each row owns its object: a `FILE` column is strictly one object per row, and a `DIRECTORY` column is a per-row immutable subtree that is replaced as a whole.

## Accessing objects from procedures

Guests address content with two values: the object `oid` and an object-relative `path` (empty or `.` for `FILE` objects; an entry path for `DIRECTORY` objects). Paths are normalized inside the object: `..` escaping the object root is rejected with `PermissionDenied`, and absolute or otherwise illegal components with `InvalidFilename`.

```rust
// Write: open a PENDING object created by the current transaction.
let fd = mudu_fs_open(session, photo_oid, "", O_WRONLY)?;
mudu_fs_write(session, fd, &bytes)?;
mudu_fs_close(session, fd)?; // fsync and seal the private generation

// Read: anchor the generation visible at open time.
let fd = mudu_fs_open(session, photo_oid, "", O_RDONLY)?;
let data = mudu_fs_read(session, fd, 4096)?;
mudu_fs_close(session, fd)?;
```

- Open flags use libc `O_*` values (`O_RDONLY` / `O_WRONLY` / `O_RDWR`). `O_CREAT`, `O_TRUNC`, `O_APPEND`, and `O_EXCL` are rejected (`InvalidInput`): object creation goes through FS-column DML, and content replacement goes through a new generation.
- A write open is allowed only on a PENDING object created by the current transaction; it writes a new private generation that is invisible to other transactions. `mudu_fs_close` fsyncs the content and seals the generation; committing the surrounding transaction publishes it, rolling back discards it. Committed generations are read-only.
- A read open anchors the generation visible at open time, so the content does not change for the lifetime of the fd.
- File descriptors are session-scoped and are released when the session closes; operating on a closed fd fails with `BadFileDescriptor`.

## Filesystem syscalls

The fs family has eleven syscalls, exposed through `sys_interface::sync_api` and `sys_interface::async_api` under the same names:

| Function | Semantics |
|----------|-----------|
| `mudu_fs_open(session_id, oid, path, flags) -> RS<u32>` | Open an object (or entry) and return an fd. |
| `mudu_fs_close(session_id, fd) -> RS<()>` | Close an fd; closing a write fd seals its private generation. |
| `mudu_fs_read(session_id, fd, len) -> RS<Vec<u8>>` | Read at the fd cursor, advancing it. |
| `mudu_fs_write(session_id, fd, data) -> RS<u32>` | Write at the fd cursor, advancing it; returns bytes written. |
| `mudu_fs_pread(session_id, fd, offset, len) -> RS<Vec<u8>>` | Read at `offset` without moving the cursor. |
| `mudu_fs_pwrite(session_id, fd, offset, data) -> RS<()>` | Write at `offset` without moving the cursor. |
| `mudu_fs_lseek(session_id, fd, offset, whence) -> RS<u64>` | Move the cursor (`whence` 0/1/2 = SET/CUR/END); returns the new cursor. |
| `mudu_fs_fstat(session_id, fd) -> RS<FsStat>` | Stat an open fd. |
| `mudu_fs_stat(session_id, oid, path) -> RS<FsStat>` | Stat an object or entry without opening an fd. |
| `mudu_fs_fsync(session_id, fd) -> RS<()>` | Flush a write fd's content to durable storage. |
| `mudu_fs_readdir(session_id, oid, path) -> RS<Vec<FsDirEntry>>` | List the entries of a DIRECTORY object. |

`FsStat` carries `{ oid, generation, entry, length, state }` (`state`: 0 = PENDING, 1 = SEALED); `FsDirEntry` carries `{ name, is_dir, length }`.

- A single `mudu_fs_read` / `mudu_fs_write` transfers at most 16 MiB; larger requests fail with `InvalidInput`.
- `mtp rust --async` converts these calls like any other syscall: import them by bare name from `sys_interface::sync_api` so the transpiler can rewrite them.

## Generations and garbage collection

- Object content lives under `{data_dir}/fs/{fs_id}` as `{oidhex}.{generation}[.{entry}]` files, where `oidhex` is the 32-character lowercase hex of the object OID.
- Writing an object produces a whole new generation. Committed generations are immutable, so content bytes never enter the WAL — only `_fs_object` metadata rows do.
- At startup, after WAL replay, a recovery scan deletes generations that no visible, sealed `_fs_object` row references.
- A periodic GC reclaims generations once they are no longer visible to the oldest active snapshot: deleting a row or rebinding its FS column makes the old generation collectable after it passes that horizon.
- GC and storage are per worker: each worker scans only its own storage root.

## Standalone adapters

On the standalone SQLite / PostgreSQL / MySQL adapter paths, the fs syscalls are emulated on the local filesystem under `{db_path}.fs/` with the same flat layout, pinned to a single generation (always `1`, with state reported as SEALED) and without MVCC, DDL, or catalog support. The standalone `mudud` driver path returns `NotImplemented`: fs frames are not yet wired into the mudud wire protocol.

Filesystem content is stored outside WAL. The current worker snapshot copies relation `.dat` files and KV state, not the `fs/` generation files; logical dumps contain schema and table CSV rather than object-content files. These mechanisms alone therefore do not provide complete replication or recovery of filesystem object content. Preserve and validate that content separately when using FS columns.
