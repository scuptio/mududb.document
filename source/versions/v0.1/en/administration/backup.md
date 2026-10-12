# Backup and Restore

MuduDB provides two backup mechanisms:

- **Base backup + WAL archiving** — physical snapshots of every worker at a committed point, plus continuously archived WAL chunks for point-in-time recovery (PITR).
- **Logical dump** — a portable full-database export (schema plus per-table CSV) packaged as a ZIP archive.

Base backups and logical dumps run on the live server through the HTTP management endpoint. `restore` and `pitr` are offline commands that build a new data directory locally without contacting the server.

## Configuration

Backup is configured through the optional `[backup]` section of `mudud.cfg`. When the section is absent the server runs backup-free: WAL prefix GC reclaims chunks without archiving them.

```toml
[backup]
backup_dir = "./backup"         # base backups are written here
archive_dir = "./wal-archive"   # sealed WAL chunks are archived here
archive_mode = "on"             # "off", "on" (default), or "always"
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `backup_dir` | string | unset | Directory base backups are written to; required by the `mcli backup` commands. |
| `archive_dir` | string | unset | Directory sealed WAL chunks are continuously archived to. |
| `archive_mode` | string | `"on"` | `"off"` disables archiving; `"on"` archives sealed chunks on a periodic cadence; `"always"` requires the archive copy to be durable before a commit is acknowledged (not yet wired into the commit path). Archiving is active only when `archive_dir` is set and the mode is not `"off"`; an unknown mode fails startup. |

When archiving is active, WAL prefix GC only reclaims a chunk after it has been durably archived. Archiving runs independently of replication, so single-node deployments archive too.

## Taking a base backup

```bash
mcli --http-addr 127.0.0.1:8300 backup create --label daily-1
```

The server force-flushes each worker's WAL, snapshots every worker at its committed point, and copies the snapshots under `<backup_dir>/<label>/`. The `BACKUP_LABEL` file is written last: a backup counts as valid only once its label decodes and every referenced snapshot is complete.

List, verify, and delete backups:

```bash
mcli --http-addr 127.0.0.1:8300 backup list
mcli --http-addr 127.0.0.1:8300 backup verify --label daily-1
mcli --http-addr 127.0.0.1:8300 backup delete --label daily-1
```

`backup list` returns the labels of every valid backup under `backup_dir`. `backup verify` decodes the label and checks every referenced snapshot file against its manifest (length and CRC64); a missing or corrupt backup fails with an error. `backup delete` removes `<backup_dir>/<label>/`.

All four commands require `[backup] backup_dir`; without it the server answers `backup is not configured: [backup] backup_dir is missing`.

## Restoring a base backup (offline)

```bash
mcli restore --label daily-1 --backup-dir ./backup --archive-dir ./wal-archive --target-dir ./restored
```

`restore` rebuilds a complete data directory under `--target-dir`: it installs the base backup's relation files and key/value state, then replays the archived WAL forward to the end of the archive. `--backup-dir` and `--archive-dir` are the directories `[backup] backup_dir` and `archive_dir` were configured with. A server can then be started with `db_path` pointing at the target directory.

## Point-in-time recovery (offline)

```bash
mcli pitr --label daily-1 --backup-dir ./backup --archive-dir ./wal-archive --target-dir ./recovered --time 1700000000000
```

`pitr` works like `restore` but stops at a target point. Exactly one of the following is required:

- `--lsn <N>` — stop before the given LSN.
- `--time <ms>` — stop after the last commit whose timestamp is at or before the given milliseconds since the epoch.
- `--xid <N>` — stop after the commit of the given transaction id.

If no commit in the archive satisfies a `--time` or `--xid` target, the command fails with `no commit in the archive satisfies the recovery target`.

## Logical dump and restore

```bash
mcli --http-addr 127.0.0.1:8300 dump --label daily-1 --dir /dumps
```

Writes a self-contained ZIP archive `<dir>/<label>.zip` on the server (`--dir` is a server-side path). The archive contains the catalog metadata (`manifest.mp`, `schema.mp`), a human-readable `schema.sql`, and one CSV per table under `data/` (CSV encoding is identical to the SQL `COPY` surface). Partitioned tables contribute metadata only; their data is not dumped.

```bash
mcli --http-addr 127.0.0.1:8300 restore-logical --label daily-1 --dir /dumps
mcli --http-addr 127.0.0.1:8300 restore-logical --label daily-1 --dir /dumps --drop
```

Restores the schema and reloads the data on the live server. Tables that already exist are skipped by default; pass `--drop` to drop and recreate same-named tables.

## HTTP endpoints

The `mcli` commands map to these HTTP management endpoints:

| Command | Endpoint |
|---------|----------|
| `mcli backup create --label L` | `POST /mudu/backup` |
| `mcli backup list` | `GET /mudu/backup/list` |
| `mcli backup verify --label L` | `POST /mudu/backup/verify` |
| `mcli backup delete --label L` | `POST /mudu/backup/delete` |
| `mcli dump --label L --dir D` | `POST /mudu/dump` |
| `mcli restore-logical --label L --dir D` | `POST /mudu/restore-logical` |

## Limitations

- Base backup and logical dump require a kernel server mode (`Tokio` or `IOUring`). On a `Legacy` server the endpoints report errors such as `base backup is not supported`.
- A base backup covers one server. `restore` and `pitr` run locally against its `backup_dir` and `archive_dir` and do not need a running server.

See the [Configuration Reference](configuration.md) for all `[backup]` fields and [mcli Management Interface (HTTP)](../reference/mcli-admin.md) for the full command reference.

WAL archiving from `[backup]` is wired into the `Tokio` backend's worker configuration. The current `IOUring` startup path does not pass that configuration to workers: its base-backup and logical-dump management endpoints do not imply continuous WAL archiving. Use `Tokio` for the documented archive/PITR workflow. `archive_mode = "always"` currently does not gate commit acknowledgements on archive durability.

Filesystem content is stored outside WAL. The current worker snapshot copies relation `.dat` files and KV state, not the `fs/` generation files; logical dumps contain schema and table CSV rather than object-content files. These mechanisms alone therefore do not provide complete replication or recovery of filesystem object content. Preserve and validate that content separately when using FS columns.
