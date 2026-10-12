# 配置参考

`mudud` 从 TOML 配置文件读取配置。该文件控制监听端口、执行模式、存储路径与 io_uring 行为。

## 文件位置

显式指定 `--cfg` / `-c` 时只读取该路径；文件缺失或无效会报错，不会回退查找其他配置。未指定时依次查找 `./mudud.cfg`、`~/.mududb/mudud.cfg`；都不存在则返回 `NotFound`。相对存储路径以服务端进程的工作目录为基准。

## 生成默认配置

```bash
mudud init-cfg
```

## 配置字段

下表列出生成配置的取值及可选字段默认值。手写 TOML 必须包含 `mpk_path`、`db_path`、`listen_ip`、`http_listen_port`、`pg_listen_port` 和 `enable_async`，解析器不会为它们补默认值。`mudud init-cfg` 写入 `./mudud.cfg`，并会覆盖已有文件。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `mpk_path` | string | `"./mpk"` | 应用包目录路径。 |
| `db_path` | string | `"./data"` | 数据库文件目录路径。 |
| `listen_ip` | string | `127.0.0.1` | 监听 IP 地址。 |
| `http_listen_port` | u16 | `8300` | HTTP 管理 API 端口。 |
| `http_worker_threads` | usize | `1` | HTTP worker 线程数。 |
| `pg_listen_port` | u16 | `5432` | PostgreSQL wire protocol 端口（仅 Legacy）。 |
| `component_target` | string | `p2` | Wasm 组件 ABI 目标。允许值：`p2`、`p3`。 |
| `enable_async` | boolean | `true` | 是否启用 WASI 组件运行时。 |
| `server_mode` | string | `"Tokio"` | `"Legacy"`、`"IOUring"` 或 `"Tokio"`。 |
| `tcp_listen_port` | u16 | `9527` | TCP 定帧协议端口。 |
| `tcp_multi_port` | boolean | `false` | 每个 worker 一个 TCP 监听器。 |
| `worker_threads` | usize | `0` | Worker 线程数。`0` 表示使用可用并行度。 |
| `io_uring_ring_entries` | u32 | `1024` | io_uring 完成队列深度。 |
| `io_uring_accept_multishot` | boolean | `true` | 启用 io_uring accept multishot。 |
| `io_uring_recv_multishot` | boolean | `true` | 启用 io_uring recv multishot。 |
| `io_uring_enable_fixed_buffers` | boolean | `false` | 启用 io_uring fixed buffers。 |
| `io_uring_enable_fixed_files` | boolean | `false` | 启用 io_uring fixed files。 |
| `routing_mode` | string | `"ConnectionId"` | `"ConnectionId"`、`"PlayerId"` 或 `"RemoteHash"`。 |
| `log_chunk_size` | u64 | `64 * 1024 * 1024` | io_uring 日志 chunk 大小，单位字节。 |
| `page_size` | usize | `4096` | 数据库页大小，单位字节。持久化配置；已有数据库变更该值需要迁移或重新初始化。 |
| `wal_flush_max_wait_us` | u64 | `200` | 组提交攒批窗口，单位微秒：排队中的 WAL 提交批在被 flush 驱动冲刷前可等待的时长。 |
| `wal_sync_mode` | string | `"commit"` | WAL 持久化模式：`"commit"` 在确认提交前对每个 flush 轮次做 fsync；`"periodic"` 在 WAL 写入进入页缓存后即确认，每个 `wal_sync_interval_ms` 至多 fsync 一次。未知取值会导致启动失败。 |
| `wal_sync_interval_ms` | u64 | `10` | `wal_sync_mode = "periodic"` 时的 WAL fsync 间隔，单位毫秒。掉电可能丢失最近一个间隔内已确认的提交。 |

## 服务端与路由模式

- `server_mode`：控制后端 I/O/运行时路径。
  - `Legacy`：传统阻塞 I/O 路径，兼容性最好。
  - `IOUring`：基于 Linux io_uring 的高性能异步 I/O。
  - `Tokio`：基于 Tokio 的异步运行时。

- `routing_mode`：控制请求如何路由到工作线程。
  - `ConnectionId`（默认）：基于连接标识符路由。
  - `PlayerId`：基于应用层玩家/用户标识符路由。
  - `RemoteHash`：基于远程地址或哈希键路由。

## 过程资源限制

可选的 `[invocation_limits]` 表限制 WASM 过程执行。省略的字段使用下列默认值；修改后需要重启。所有取值必须大于零，`memory_bytes` 不能超过 `total_memory_bytes`，`max_concurrent_per_app` 不能超过 `max_concurrent`。该表中的未知字段会被拒绝。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `memory_bytes` | usize | `134217728` | 单次调用中所有内存实例的线性内存总字节数。 |
| `fuel` | u64 | `1000000000` | Wasm 指令燃料，含组件初始化。 |
| `timeout_ms` | u64 | `30000` | 墙钟执行预算，单位毫秒，含事务建立。 |
| `max_input_bytes` | usize | `8388608` | 编码后的过程或 host 调用输入大小上限。 |
| `max_output_bytes` | usize | `8388608` | 编码后的过程或 host 调用输出大小上限。 |
| `max_host_calls` | u64 | `10000` | host 调用次数上限，含打开资源的调用。 |
| `max_resources` | usize | `256` | 同时打开的 worker 会话与文件句柄上限。 |
| `table_elements` | usize | `100000` | 单次调用中表元素总数上限。 |
| `max_concurrent` | usize | `4` | 进程级获准并发过程调用数上限。 |
| `max_concurrent_per_app` | usize | `2` | 同名应用的获准并发调用数上限。 |
| `total_memory_bytes` | usize | `536870912` | 进程级预留 guest 线性内存预算。 |

## 复制

可选的 `[replication]` 表配置主从流复制或 Raft 容错。缺省时服务器以单机运行。复制目前要求 `server_mode = "Tokio"`；`Legacy` 和 `IOUring` 均会拒绝启动。`[replication.stream]` 与 `[replication.raft]` 子段必须且只能存在一个——存在的子段即模式。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `node_id` | u64 | 必填 | 本节点标识。首次启动时持久化到 `<db_path>/node_id`；与持久化值不一致会导致启动失败。 |
| `listen_port` | u16 | 必填 | 复制与 Raft 通道监听端口；不得与 `http_listen_port`、`pg_listen_port`、`tcp_listen_port` 冲突。 |
| `repl_flow_window_segments` | u64 | `64` | 复制流的流控窗口，单位为段。 |
| `snapshot_wal_threshold_bytes` | u64 | `1073741824` | 距上次快照后已提交 WAL 字节数超过该值时触发下一次按 worker 快照。必须大于零。 |
| `gc_check_interval_secs` | u64 | `60` | WAL/暂存 GC 水位评估间隔，单位秒。必须大于零。 |

`[replication.stream]` —— 主从流复制（无 Raft）：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `primary` | boolean | 必填 | 本节点是否为配置的 primary；集群中恰好一个成员设为 `true`。 |
| `sync` | boolean | 必填 | primary 是否等待某个副本的持久化确认后再应答提交。 |
| `sync_timeout_ms` | u64 | `10000` | 同步复制等待超时，单位毫秒。 |

`[replication.raft]` —— Raft 共识容错：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `raft_tick_ms` | u64 | `50` | Raft 状态机 tick 间隔，单位毫秒。 |
| `election_timeout_ticks` | u64 | `500` | Raft 选举超时，单位为 tick。 |

集群成员以 `[[replication.peers]]` 条目列出，含 `node_id`、`host`、`port`。peer 列表必须非空、节点 ID 与地址不重复，且包含本节点的 `node_id`。流复制模式要求至少两个 peer。

## 备份

可选的 `[backup]` 表配置基础备份与 WAL 归档。缺省时服务器以无备份方式运行：WAL 前缀 GC 直接回收 chunk，不做归档。备份工作流程见[备份与恢复](backup.md)。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `backup_dir` | string | 未设置 | 基础备份的写入目录；`mcli backup` 命令要求配置。 |
| `archive_dir` | string | 未设置 | 已封口 WAL chunk 持续归档到的目录。 |
| `archive_mode` | string | `"on"` | `"off"`、`"on"` 或 `"always"`。仅当设置了 `archive_dir` 且模式不是 `"off"` 时归档生效；未知模式会导致启动失败。 |

## 示例 `mudud.cfg`

```toml
mpk_path = "./mpk"
db_path = "./data"
listen_ip = "127.0.0.1"
http_listen_port = 8300
http_worker_threads = 1
pg_listen_port = 5432
tcp_listen_port = 9527
server_mode = "Tokio"
worker_threads = 0
io_uring_ring_entries = 1024
io_uring_accept_multishot = true
io_uring_recv_multishot = true
io_uring_enable_fixed_buffers = false
io_uring_enable_fixed_files = false
routing_mode = "ConnectionId"
enable_async = true
tcp_multi_port = false
log_chunk_size = 67108864
page_size = 4096
```

兼容性与升级规则请参见[服务端配置契约](../reference/contracts/mudud_cfg_v1.md)。

`[backup]` 的 WAL 归档配置接入了 `Tokio` 后端的 worker 配置。当前 `IOUring` 启动路径没有将该配置传给 worker；其基础备份和逻辑导出管理端点不代表持续 WAL 归档已启用。本文的归档/PITR 流程应使用 `Tokio`。`archive_mode = "always"` 当前尚未将归档持久化接入提交确认门控。

完整集群示例见[复制与 Raft 容错](replication.md)，各模式的监听端点见[服务端管理](server.md)。
