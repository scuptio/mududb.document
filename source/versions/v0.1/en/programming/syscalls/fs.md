# fs

The filesystem syscalls provide POSIX-like file operations on filesystem objects (fs objects), with 11 syscalls in total.

## Rust API

```rust
// sync_api (async_api entries have the same names as async fns)
pub fn mudu_fs_open(session_id: OID, oid: OID, path: &str, flags: u32) -> RS<u32> { /* ... */ }
pub fn mudu_fs_close(session_id: OID, fd: u32) -> RS<()> { /* ... */ }
pub fn mudu_fs_read(session_id: OID, fd: u32, len: u32) -> RS<Vec<u8>> { /* ... */ }
pub fn mudu_fs_write(session_id: OID, fd: u32, data: &[u8]) -> RS<u32> { /* ... */ }
pub fn mudu_fs_pread(session_id: OID, fd: u32, offset: u64, len: u32) -> RS<Vec<u8>> { /* ... */ }
pub fn mudu_fs_pwrite(session_id: OID, fd: u32, offset: u64, data: &[u8]) -> RS<()> { /* ... */ }
pub fn mudu_fs_lseek(session_id: OID, fd: u32, offset: i64, whence: u32) -> RS<u64> { /* ... */ }
pub fn mudu_fs_fstat(session_id: OID, fd: u32) -> RS<FsStat> { /* ... */ }
pub fn mudu_fs_stat(session_id: OID, oid: OID, path: &str) -> RS<FsStat> { /* ... */ }
pub fn mudu_fs_fsync(session_id: OID, fd: u32) -> RS<()> { /* ... */ }
pub fn mudu_fs_readdir(session_id: OID, oid: OID, path: &str) -> RS<Vec<FsDirEntry>> { /* ... */ }
```

## Parameters

- `oid` — OID of the filesystem object.
- `path` — relative path within the object.
- `fd` — file descriptor returned by `mudu_fs_open`.

## Behavior

- `mudu_fs_open` opens the file at `path` inside the filesystem object identified by `oid` and returns a file descriptor `fd`; `flags` takes libc `O_*` values.
- `mudu_fs_read` / `mudu_fs_write` read and write at the current cursor of `fd`; `mudu_fs_pread` / `mudu_fs_pwrite` read and write at the given `offset`.
- `mudu_fs_lseek` moves the cursor of `fd`; `mudu_fs_fsync` flushes the writes on `fd` to disk; `mudu_fs_close` closes `fd`.
- `mudu_fs_fstat` / `mudu_fs_stat` return file metadata as `FsStat`; `mudu_fs_readdir` lists directory entries as `FsDirEntry` (directories only).

## mtp Sync-to-Async Constraints

`mtp rust --async` automatically converts all syscalls (including the `mudu_fs_*` family) in a `/**mudu-proc**/` procedure into async form. Import them via `use sys_interface::sync_api::{mudu_fs_open, ...}` and call by bare name for mtp to recognize and convert them; calls qualified through a module path (such as `sync_api::mudu_fs_open(...)`) are not converted.

See [Filesystem Object Types](../filesystem.md) for DDL, immutable generations, accepted flags and backend restrictions. The session argument shown here belongs to the workspace Rust facade; the low-level WIT / standalone SDK fd operations do not take that extra argument.
