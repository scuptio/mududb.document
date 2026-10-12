# 备份与恢复

MuduDB 提供两种备份机制：

- **基础备份 + WAL 归档**——在每个 worker 的已提交点上做物理快照，并持续归档 WAL chunk，用于时间点恢复（PITR）。
- **逻辑导出**——可移植的全库导出（schema 加每表 CSV），打包为 ZIP 归档。

基础备份与逻辑导出通过 HTTP 管理端点在运行中的服务器上执行。`restore` 与 `pitr` 是离线命令，在本地构建新的数据目录，不接触服务器。

## 配置

备份通过 `mudud.cfg` 的可选 `[backup]` 段配置。该段缺失时服务器以无备份方式运行：WAL 前缀 GC 直接回收 chunk，不做归档。

```toml
[backup]
backup_dir = "./backup"         # 基础备份写入此目录
archive_dir = "./wal-archive"   # 已封口的 WAL chunk 归档到此目录
archive_mode = "on"             # "off"、"on"（默认）或 "always"
```

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `backup_dir` | string | 未设置 | 基础备份的写入目录；`mcli backup` 命令要求配置。 |
| `archive_dir` | string | 未设置 | 已封口 WAL chunk 持续归档到的目录。 |
| `archive_mode` | string | `"on"` | `"off"` 关闭归档；`"on"` 按周期归档已封口 chunk；`"always"` 要求归档副本持久化后才确认提交（尚未接入提交路径）。仅当设置了 `archive_dir` 且模式不是 `"off"` 时归档生效；未知模式会导致启动失败。 |

归档生效时，WAL 前缀 GC 只有在 chunk 持久归档后才会回收它。归档独立于复制运行，单机部署也会归档。

## 执行基础备份

```bash
mcli --http-addr 127.0.0.1:8300 backup create --label daily-1
```

服务器强制冲刷每个 worker 的 WAL，在其已提交点上快照每个 worker，并将快照复制到 `<backup_dir>/<label>/`。`BACKUP_LABEL` 文件最后写入：只有当标签可解码且每个引用的快照都完整时，备份才算有效。

列出、校验与删除备份：

```bash
mcli --http-addr 127.0.0.1:8300 backup list
mcli --http-addr 127.0.0.1:8300 backup verify --label daily-1
mcli --http-addr 127.0.0.1:8300 backup delete --label daily-1
```

`backup list` 返回 `backup_dir` 下每个有效备份的标签。`backup verify` 解码标签并依据清单（长度与 CRC64）检查每个引用的快照文件；备份缺失或损坏会报错。`backup delete` 删除 `<backup_dir>/<label>/`。

这四个命令都要求 `[backup] backup_dir`；未配置时服务器返回 `backup is not configured: [backup] backup_dir is missing`。

## 恢复基础备份（离线）

```bash
mcli restore --label daily-1 --backup-dir ./backup --archive-dir ./wal-archive --target-dir ./restored
```

`restore` 在 `--target-dir` 下重建完整数据目录：安装基础备份的关系文件与键值状态，然后将归档 WAL 向前重放到归档末尾。`--backup-dir` 与 `--archive-dir` 即 `[backup] backup_dir` 与 `archive_dir` 配置的目录。之后可将 `db_path` 指向该目标目录启动服务器。

## 时间点恢复（离线）

```bash
mcli pitr --label daily-1 --backup-dir ./backup --archive-dir ./wal-archive --target-dir ./recovered --time 1700000000000
```

`pitr` 与 `restore` 类似，但在目标点停止。必须且只能提供以下之一：

- `--lsn <N>` —— 在给定 LSN 之前停止。
- `--time <ms>` —— 停止于时间戳不晚于给定毫秒值（自 epoch 起）的最后一个提交之后。
- `--xid <N>` —— 在给定事务 ID 的提交之后停止。

如果归档中没有提交满足 `--time` 或 `--xid` 目标，命令以 `no commit in the archive satisfies the recovery target` 报错。

## 逻辑导出与导入

```bash
mcli --http-addr 127.0.0.1:8300 dump --label daily-1 --dir /dumps
```

在服务器上写出自包含的 ZIP 归档 `<dir>/<label>.zip`（`--dir` 是服务器端路径）。归档包含目录元数据（`manifest.mp`、`schema.mp`）、可读的 `schema.sql`，以及 `data/` 下每表一个 CSV（CSV 编码与 SQL `COPY` 接口一致）。分区表只导出元数据，不导出数据。

```bash
mcli --http-addr 127.0.0.1:8300 restore-logical --label daily-1 --dir /dumps
mcli --http-addr 127.0.0.1:8300 restore-logical --label daily-1 --dir /dumps --drop
```

在运行中的服务器上恢复 schema 并重新加载数据。默认跳过已存在的同名表；传入 `--drop` 则删除并重建同名表。

## HTTP 端点

`mcli` 命令与 HTTP 管理端点的对应关系：

| 命令 | 端点 |
|------|------|
| `mcli backup create --label L` | `POST /mudu/backup` |
| `mcli backup list` | `GET /mudu/backup/list` |
| `mcli backup verify --label L` | `POST /mudu/backup/verify` |
| `mcli backup delete --label L` | `POST /mudu/backup/delete` |
| `mcli dump --label L --dir D` | `POST /mudu/dump` |
| `mcli restore-logical --label L --dir D` | `POST /mudu/restore-logical` |

## 限制

- 基础备份与逻辑导出要求内核服务端模式（`Tokio` 或 `IOUring`）。在 `Legacy` 服务器上这些端点会报错，如 `base backup is not supported`。
- 基础备份覆盖单台服务器。`restore` 与 `pitr` 在本地针对其 `backup_dir` 与 `archive_dir` 运行，不需要运行中的服务器。

全部 `[backup]` 字段请参见[配置参考](configuration.md)，完整命令参考请参见 [mcli 管理接口（HTTP）](../reference/mcli-admin.md)。
