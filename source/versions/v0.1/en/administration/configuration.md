# Configuration Reference

`mudud` reads its configuration from a TOML file. The file controls listen ports, execution mode, storage paths, and io_uring behavior.

## File location

An explicit `--cfg` / `-c` path is used exclusively: a missing or invalid file fails without fallback. Without that option, the server tries `./mudud.cfg`, then `~/.mududb/mudud.cfg`; if neither exists it returns `NotFound`. Relative storage paths are resolved from the server process working directory.

## Generate a default config

```bash
mudud init-cfg
```

## Configuration fields

The table lists generated configuration values and defaults for optional fields. `mpk_path`, `db_path`, `listen_ip`, `http_listen_port`, `pg_listen_port`, and `enable_async` are required in a hand-written TOML file; the parser does not fill them in. `mudud init-cfg` writes `./mudud.cfg` and overwrites an existing file.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `mpk_path` | string | `"./mpk"` | Path to the application package directory. |
| `db_path` | string | `"./data"` | Path to the database directory. |
| `listen_ip` | string | `127.0.0.1` | IP address to listen on. |
| `http_listen_port` | u16 | `8300` | HTTP management API port. |
| `http_worker_threads` | usize | `1` | HTTP worker thread count. |
| `pg_listen_port` | u16 | `5432` | PostgreSQL wire protocol port (Legacy only). |
| `component_target` | string | `p2` | Wasm component ABI target. Allowed: `p2`, `p3`. |
| `enable_async` | boolean | `true` | Enable the WASI component runtime. |
| `server_mode` | string | `"Tokio"` | `"Legacy"`, `"IOUring"`, or `"Tokio"`. |
| `tcp_listen_port` | u16 | `9527` | TCP framed protocol port. |
| `tcp_multi_port` | boolean | `false` | One TCP listener per worker. |
| `worker_threads` | usize | `0` | Worker thread count. `0` means use available parallelism. |
| `io_uring_ring_entries` | u32 | `1024` | io_uring completion queue depth. |
| `io_uring_accept_multishot` | boolean | `true` | Enable io_uring accept multishot. |
| `io_uring_recv_multishot` | boolean | `true` | Enable io_uring recv multishot. |
| `io_uring_enable_fixed_buffers` | boolean | `false` | Enable io_uring fixed buffers. |
| `io_uring_enable_fixed_files` | boolean | `false` | Enable io_uring fixed files. |
| `routing_mode` | string | `"ConnectionId"` | `"ConnectionId"`, `"PlayerId"`, or `"RemoteHash"`. |
| `log_chunk_size` | u64 | `64 * 1024 * 1024` | io_uring log chunk size in bytes. |
| `page_size` | usize | `4096` | Database page size in bytes. Persistent: changing it for an existing database requires migration or re-initialization. |
| `wal_flush_max_wait_us` | u64 | `200` | Group-commit batching window in microseconds: how long a queued WAL commit batch may age before the flush driver flushes it. |
| `wal_sync_mode` | string | `"commit"` | WAL durability mode: `"commit"` fsyncs every flush round before the commit is acknowledged; `"periodic"` acknowledges once the WAL write lands in the page cache and fsyncs at most once per `wal_sync_interval_ms`. Unknown values fail startup. |
| `wal_sync_interval_ms` | u64 | `10` | WAL fsync interval in milliseconds for `wal_sync_mode = "periodic"`. A power loss may lose acknowledged commits from the last interval. |

## Server and routing modes

- `server_mode`: controls the backend I/O/runtime path.
  - `Legacy`: traditional blocking I/O path for maximum compatibility.
  - `IOUring`: Linux io_uring based high-performance async I/O.
  - `Tokio`: Tokio-based async runtime.

- `routing_mode`: controls how requests are routed to workers.
  - `ConnectionId` (default): route based on connection identifier.
  - `PlayerId`: route based on an application-level player/user identifier.
  - `RemoteHash`: hash-based routing using remote address or hashed key.

## Procedure resource limits

The optional `[invocation_limits]` table caps WASM procedure execution. Omitted fields use the defaults below; changes require a restart. Every value must be positive, `memory_bytes` must not exceed `total_memory_bytes`, and `max_concurrent_per_app` must not exceed `max_concurrent`. Unknown fields in this table are rejected.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `memory_bytes` | usize | `134217728` | Total linear-memory bytes across all memories in one invocation. |
| `fuel` | u64 | `1000000000` | Wasm instruction fuel, including component initialization. |
| `timeout_ms` | u64 | `30000` | Wall-clock execution budget in milliseconds, including transaction setup. |
| `max_input_bytes` | usize | `8388608` | Maximum encoded procedure or host-call input size. |
| `max_output_bytes` | usize | `8388608` | Maximum encoded procedure or host-call output size. |
| `max_host_calls` | u64 | `10000` | Maximum host calls, including resource-opening calls. |
| `max_resources` | usize | `256` | Maximum simultaneously open worker sessions and file handles. |
| `table_elements` | usize | `100000` | Maximum total table elements in one invocation. |
| `max_concurrent` | usize | `4` | Process-wide maximum admitted procedure calls. |
| `max_concurrent_per_app` | usize | `2` | Maximum admitted calls sharing an application name. |
| `total_memory_bytes` | usize | `536870912` | Process-wide reserved guest linear-memory budget. |

## Replication

The optional `[replication]` table configures primary-replica streaming replication or Raft fault tolerance. When absent, the server runs as a single node. Replication currently requires `server_mode = "Tokio"`; `Legacy` and `IOUring` reject it at startup. Exactly one of the `[replication.stream]` / `[replication.raft]` subsections must be present — the present subsection selects the mode.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `node_id` | u64 | required | Identifier of this node. Persisted to `<db_path>/node_id` on first boot; a mismatch with the persisted value is a startup error. |
| `listen_port` | u16 | required | Port the replication and Raft channel listens on; must not collide with `http_listen_port`, `pg_listen_port`, or `tcp_listen_port`. |
| `repl_flow_window_segments` | u64 | `64` | Flow-control window of the replication stream, in segments. |
| `snapshot_wal_threshold_bytes` | u64 | `1073741824` | Committed WAL bytes beyond the last snapshot that trigger the next per-worker snapshot. Must be greater than zero. |
| `gc_check_interval_secs` | u64 | `60` | Interval between WAL/staging GC watermark evaluations, in seconds. Must be greater than zero. |

`[replication.stream]` — primary-replica streaming replication without Raft:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `primary` | boolean | required | Whether this node is the configured primary; exactly one cluster member sets `true`. |
| `sync` | boolean | required | Whether the primary waits for a replica's durable ack before answering a commit. |
| `sync_timeout_ms` | u64 | `10000` | Synchronous replication wait timeout in milliseconds. |

`[replication.raft]` — Raft consensus fault tolerance:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `raft_tick_ms` | u64 | `50` | Raft state machine tick interval in milliseconds. |
| `election_timeout_ticks` | u64 | `500` | Raft election timeout in ticks. |

Cluster members are listed as `[[replication.peers]]` entries with `node_id`, `host`, and `port`. The peer list must be non-empty, contain no duplicate node ids or addresses, and include this node's own `node_id`. Stream mode requires at least two peers.

## Backup

The optional `[backup]` table configures base backups and WAL archiving. When absent, the server runs backup-free: WAL prefix GC reclaims chunks without archiving them. See [Backup and Restore](backup.md) for the backup workflow.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `backup_dir` | string | unset | Directory base backups are written to; required by the `mcli backup` commands. |
| `archive_dir` | string | unset | Directory sealed WAL chunks are continuously archived to. |
| `archive_mode` | string | `"on"` | `"off"`, `"on"`, or `"always"`. Archiving is active only when `archive_dir` is set and the mode is not `"off"`; an unknown mode fails startup. |

## Example `mudud.cfg`

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

See the [Server Configuration Contract](../reference/contracts/mudud_cfg_v1.md) for compatibility and upgrade rules.

WAL archiving from `[backup]` is wired into the `Tokio` backend's worker configuration. The current `IOUring` startup path does not pass that configuration to workers: its base-backup and logical-dump management endpoints do not imply continuous WAL archiving. Use `Tokio` for the documented archive/PITR workflow. `archive_mode = "always"` currently does not gate commit acknowledgements on archive durability.

See [Replication and Raft Fault Tolerance](replication.md) for complete cluster examples and [Server Management](server.md) for listeners by mode.
