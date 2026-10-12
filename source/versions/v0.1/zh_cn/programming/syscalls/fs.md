# fs

文件系统系统调用提供对文件系统对象（fs objects）的 POSIX 风格文件操作，共 11 个系统调用。

## Rust API

```rust
// sync_api（async_api 条目同名，为 async fn）
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

## 参数

- `oid` —— 文件系统对象的对象 ID。
- `path` —— 对象内的相对路径。
- `fd` —— `mudu_fs_open` 返回的文件描述符。

## 行为

- `mudu_fs_open` 打开由 `oid` 标识的文件系统对象中 `path` 处的文件，返回文件描述符 `fd`；`flags` 取 libc `O_*` 值。
- `mudu_fs_read` / `mudu_fs_write` 在 `fd` 当前游标处读写；`mudu_fs_pread` / `mudu_fs_pwrite` 在给定 `offset` 处读写。
- `mudu_fs_lseek` 移动 `fd` 的游标；`mudu_fs_fsync` 将 `fd` 的写落盘；`mudu_fs_close` 关闭 `fd`。
- `mudu_fs_fstat` / `mudu_fs_stat` 以 `FsStat` 返回文件元数据；`mudu_fs_readdir` 以 `FsDirEntry` 列出目录项（仅目录）。

## mtp 同步转异步约束

`mtp rust --async` 会把 `/**mudu-proc**/` 过程中所有系统调用（含 `mudu_fs_*` 家族）自动转为异步形式。请以裸名 `use sys_interface::sync_api::{mudu_fs_open, ...}` 导入并调用，mtp 才能识别并转换；通过模块路径限定调用（如 `sync_api::mudu_fs_open(...)`）不会被转换。

对象 DDL、不可变 generation、允许的 flags 与后端限制见[文件系统对象类型](../filesystem.md)。这里的会话参数属于仓库内 Rust facade；底层 WIT / 独立 SDK 的 fd 操作没有该额外参数。
