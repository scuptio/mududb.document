# 文件系统对象类型

MuduDB 可以通过**文件系统（fs）对象类型**在数据库中存储文件内容。fs 对象类型由 DDL 注册，其对象的全部存储都由数据库组织和管理。每个对象以 OID 标识，内容存储为不可变的 *generation* 文件，元数据保存在 `_fs_object` 系统表中，与普通表共享同一套并发控制（MVCC）与 WAL 机制。过程通过一组 POSIX 风格的系统调用访问对象；guest 只会接触到 OID 和文件描述符，永远接触不到宿主路径。

## 注册文件系统类型

```sql
CREATE TYPE FILESYSTEM FILE photo_fs;
CREATE TYPE FILESYSTEM DIRECTORY asset_fs;
DROP TYPE photo_fs;
```

- `FILE` 为单内容对象；`DIRECTORY` 为包含一棵不可变命名条目树的对象。
- `CREATE TYPE FILESYSTEM` / `DROP TYPE` 需要 admin 会话；其他会话报 `PermissionDenied`。
- 类型重名报 `AlreadyExists`。
- 当表列仍引用该类型时，`DROP TYPE` 被拒绝（`InvalidState`）。
- 在 `CREATE TABLE` 中使用未知的 fs 类型名列报 `EntityNotFound`。

## FS 列

表通过将列类型声明为 fs 对象类型名来绑定该类型：

```sql
CREATE TABLE product (
    id BIGINT PRIMARY KEY,
    photo photo_fs
);
```

- 该列物理上存储对象 OID（`U128` 值）；catalog 记录列到 fs 类型的绑定。
- `INSERT` / `UPDATE` 时，若省略 FS 列或将其置为 `NULL`，内核会为该行创建一个新对象并自动指派其 OID（OID 最高字节带 `0xF5` 标记）。为 FS 列显式提供值会被拒绝（`InvalidArgument`）——对象只能通过 FS 列的 DML 创建。
- `SELECT` 该 FS 列返回对象 OID；可见性遵循与普通列相同的 MVCC 规则，包括本事务尚未提交的行。
- 每行拥有其对象：`FILE` 列严格一行一对象；`DIRECTORY` 列是行私有的不可变子树，替换粒度为整树。

## 在过程中访问对象

guest 用两个值寻址内容：对象 `oid` 和对象内相对路径 `path`（`FILE` 对象为空或 `.`；`DIRECTORY` 对象为条目路径）。路径在对象内做规范化：`..` 越出对象根报 `PermissionDenied`，绝对路径等非法分量报 `InvalidFilename`。

```rust
// 写：打开本事务创建的 PENDING 对象。
let fd = mudu_fs_open(session, photo_oid, "", O_WRONLY)?;
mudu_fs_write(session, fd, &bytes)?;
mudu_fs_close(session, fd)?; // fsync 并落定私有 generation

// 读：锚定打开时可见的 generation。
let fd = mudu_fs_open(session, photo_oid, "", O_RDONLY)?;
let data = mudu_fs_read(session, fd, 4096)?;
mudu_fs_close(session, fd)?;
```

- open flags 取 libc `O_*` 值（`O_RDONLY` / `O_WRONLY` / `O_RDWR`）。`O_CREAT`、`O_TRUNC`、`O_APPEND`、`O_EXCL` 被拒绝（`InvalidInput`）：对象创建走 FS 列 DML，内容替换走新 generation。
- 写打开仅允许作用于本事务创建的 PENDING 对象，写的是本事务私有的新 generation，其他事务不可见。`mudu_fs_close` 会 fsync 内容并落定 generation；外围事务提交后生效，回滚则丢弃。已提交的 generation 只读。
- 读打开锚定打开时可见的 generation，fd 存活期内内容不变。
- 文件描述符是会话作用域的，会话关闭时回收；对已关闭 fd 的操作报 `BadFileDescriptor`。

## 文件系统系统调用

fs 家族共 11 个系统调用，在 `sys_interface::sync_api` 与 `sys_interface::async_api` 下同名暴露：

| 函数 | 语义 |
|----------|-----------|
| `mudu_fs_open(session_id, oid, path, flags) -> RS<u32>` | 打开对象（或其条目），返回 fd。 |
| `mudu_fs_close(session_id, fd) -> RS<()>` | 关闭 fd；关闭写 fd 会落定其私有 generation。 |
| `mudu_fs_read(session_id, fd, len) -> RS<Vec<u8>>` | 在 fd 游标处读取并推进游标。 |
| `mudu_fs_write(session_id, fd, data) -> RS<u32>` | 在 fd 游标处写入并推进游标；返回写入字节数。 |
| `mudu_fs_pread(session_id, fd, offset, len) -> RS<Vec<u8>>` | 在 `offset` 处读取，不动游标。 |
| `mudu_fs_pwrite(session_id, fd, offset, data) -> RS<()>` | 在 `offset` 处写入，不动游标。 |
| `mudu_fs_lseek(session_id, fd, offset, whence) -> RS<u64>` | 移动游标（`whence` 0/1/2 = SET/CUR/END）；返回新游标。 |
| `mudu_fs_fstat(session_id, fd) -> RS<FsStat>` | 获取已打开 fd 的元数据。 |
| `mudu_fs_stat(session_id, oid, path) -> RS<FsStat>` | 不开 fd 获取对象或条目的元数据。 |
| `mudu_fs_fsync(session_id, fd) -> RS<()>` | 将写 fd 的内容落盘。 |
| `mudu_fs_readdir(session_id, oid, path) -> RS<Vec<FsDirEntry>>` | 列出 DIRECTORY 对象的目录项。 |

`FsStat` 携带 `{ oid, generation, entry, length, state }`（`state`：0 = PENDING，1 = SEALED）；`FsDirEntry` 携带 `{ name, is_dir, length }`。

- 单次 `mudu_fs_read` / `mudu_fs_write` 最多传输 16 MiB；超限报 `InvalidInput`。
- `mtp rust --async` 会像其他系统调用一样转换这些调用：必须经 `use sys_interface::sync_api::{...}` 以裸名导入，转译器才能识别并改写。

## Generation 与垃圾回收

- 对象内容存放在 `{data_dir}/fs/{fs_id}` 下，文件名为 `{oidhex}.{generation}[.{entry}]`，其中 `oidhex` 是对象 OID 的 32 位小写十六进制。
- 写对象会整体产生一个新 generation。已提交的 generation 不可变，因此内容字节不进入 WAL——只有 `_fs_object` 元数据行进入 WAL。
- 启动时，WAL 重放之后会执行恢复扫描，删除没有任何可见且已落定（SEALED）的 `_fs_object` 行引用的 generation。
- 周期性 GC 在 generation 越过最老活跃快照的可见性界限后将其回收：删除行或为 FS 列换绑 OID 后，旧 generation 在越过该 horizon 后可被回收。
- GC 与存储均按工作者（worker）对齐：每个 worker 只扫描自己的存储根。

## 独立适配器

在 SQLite / PostgreSQL / MySQL 的独立（standalone）适配器路径上，fs 系统调用通过本地文件系统模拟，内容位于 `{db_path}.fs/`，采用相同的扁平布局，但固定为单一 generation（恒为 `1`，state 报告为 SEALED），且不支持 MVCC、DDL 或 catalog。独立 `mudud` 驱动路径返回 `NotImplemented`：fs 帧尚未接入 mudud 线协议。

文件对象内容不进入 WAL。当前 worker 快照复制关系 `.dat` 文件和 KV 状态，不包含 `fs/` 下的 generation 文件；逻辑导出也只包含 schema 和表 CSV，不携带对象内容文件。因此仅靠这些机制不能完整复制或恢复文件对象内容。使用 FS 列时，需单独保存并验证对象内容。
