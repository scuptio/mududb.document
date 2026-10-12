# mcli Management Interface (HTTP)

This page describes the HTTP management commands available through `mcli` for application lifecycle management and partition routing queries, the replication and backup administration commands, and the TCP data-plane commands.

## Prerequisites

- A running `mudud` server.
- The HTTP management endpoint reachable (default `127.0.0.1:8300`).

Most management commands take `--http-addr <host:port>` to target the server.

## Commands

### List installed applications

```bash
mcli --http-addr 127.0.0.1:8300 app-list
```

### Install an application package

```bash
mcli --http-addr 127.0.0.1:8300 app-install --mpk target/wasm32-wasip2/release/wallet.mpk
```

### Show application procedures

```bash
mcli --http-addr 127.0.0.1:8300 app-detail --app wallet
```

### Show one procedure detail

`--proc` must be used together with `--module`.

```bash
mcli --http-addr 127.0.0.1:8300 app-detail --app wallet --module wallet --proc create_user
```

### Uninstall an application

```bash
mcli --http-addr 127.0.0.1:8300 app-uninstall --app wallet
```

### Show server topology

```bash
mcli --http-addr 127.0.0.1:8300 server-topology
```

### Partition route query

Route by exact key:

```bash
mcli --http-addr 127.0.0.1:8300 partition-route --rule-name user_rule --key user-100
```

Route by range:

```bash
mcli --http-addr 127.0.0.1:8300 partition-route --rule-name user_rule --start 100 --end 200
```

Composite key example:

```bash
mcli --http-addr 127.0.0.1:8300 partition-route --rule-name order_rule --key tenant-1,order-100
```

## Data plane commands (TCP)

These commands connect to the TCP protocol endpoint (`--addr`, default `127.0.0.1:9527`). Each takes the request body as inline JSON (`--json`) or from a JSON file (`--json-file`; use `-` to read from stdin).

The SQL example assumes an installed `demo` application with a `users(id)` table. Raw KV requests require an existing session OID as `{ "h": high_u64, "l": low_u64 }`; raw `invoke` requires `session_id` (u64 number or decimal u128 string) and encoded procedure parameters. The IDs and payloads below are placeholders, not session-creation commands. Prefer `app-invoke` for descriptor-based argument conversion and automatic session setup.

### Run a SQL query or command

```bash
mcli --addr 127.0.0.1:9527 command --json '{"app_name":"demo","sql":"SELECT id FROM users"}'
```

### Put a key-value item

```bash
mcli --addr 127.0.0.1:9527 put --json '{"oid":{"h":0,"l":1},"key":"k","value":"v"}'
```

### Get a key

```bash
mcli --addr 127.0.0.1:9527 get --json '{"oid":{"h":0,"l":1},"key":"k"}'
```

### Scan a key range

```bash
mcli --addr 127.0.0.1:9527 range --json '{"oid":{"h":0,"l":1},"start_key":"a","end_key":"z"}'
```

### Invoke a procedure with a raw request

`invoke` sends a raw procedure request over TCP. The typed variant is `app-invoke` (see [Client](../administration/client.md)), which fetches the procedure descriptor over HTTP before invoking over TCP.

```bash
mcli --addr 127.0.0.1:9527 invoke --json '{"session_id":"1","procedure_name":"app/mod/proc","procedure_parameters":{}}'
```

## Replication administration

Raft membership and primary-replica failover commands. They target the HTTP management endpoint of the node being operated on.

### Add a node to the raft group

Nodes join as voters by default; add `--learner` to join as a learner (replicates but never votes; promote later with `promote-learner`).

```bash
mcli --http-addr 127.0.0.1:8300 repl add-node --node-id 4 --host 10.0.0.4 --port 19530
mcli --http-addr 127.0.0.1:8300 repl add-node --node-id 4 --host 10.0.0.4 --port 19530 --learner
```

### Remove a node from the raft group

```bash
mcli --http-addr 127.0.0.1:8300 repl remove-node --node-id 4
```

### Promote a learner to voter

```bash
mcli --http-addr 127.0.0.1:8300 repl promote-learner --node-id 4
```

### Promote this node to stream primary

Promotion requires fencing proof: either `--fencing-cmd <cmd>`, which runs locally first (via `sh -c`) and aborts the promotion without contacting the server when it fails, or `--yes` to confirm explicitly that the old primary is fenced. The two options are mutually exclusive.

```bash
mcli --http-addr 127.0.0.1:8301 repl promote --fencing-cmd ./fence-old-primary.sh
mcli --http-addr 127.0.0.1:8301 repl promote --yes
```

### Follow a new stream primary

`--primary` is the replication address (`host:port`) of the new primary.

```bash
mcli --http-addr 127.0.0.1:8300 repl follow --primary 10.0.0.2:19527 --primary-node-id 2
```

### Show replication status

JSON by default; `--table` renders a sectioned table.

```bash
mcli --http-addr 127.0.0.1:8300 repl status
mcli --http-addr 127.0.0.1:8300 repl status --table
```

## Backup and restore

Base backups and logical dumps run on the live server through the HTTP management endpoint; `restore` and `pitr` are offline and do not contact a server. See [Backup and Restore](../administration/backup.md) for the `[backup]` configuration and the on-disk layout.

### Take a base backup

Requires `[backup] backup_dir` on the server.

```bash
mcli --http-addr 127.0.0.1:8300 backup create --label daily-1
```

### List base backups

```bash
mcli --http-addr 127.0.0.1:8300 backup list
```

### Verify a base backup

```bash
mcli --http-addr 127.0.0.1:8300 backup verify --label daily-1
```

### Delete a base backup

```bash
mcli --http-addr 127.0.0.1:8300 backup delete --label daily-1
```

### Restore a base backup (offline)

Replays the base backup plus archived WAL into `--target-dir`, up to the end of the archive.

```bash
mcli restore --label daily-1 --backup-dir /backups --archive-dir /wal-archive --target-dir /restored
```

### Point-in-time recovery (offline)

Like `restore`, but stops at a target point. Exactly one of `--lsn`, `--time`, or `--xid` is required.

```bash
mcli pitr --label daily-1 --backup-dir /backups --archive-dir /wal-archive --target-dir /recovered --time 1700000000000
```

### Take a logical dump

Writes `<dir>/<label>.zip` on the server (`--dir` is a server-side path).

```bash
mcli --http-addr 127.0.0.1:8300 dump --label daily-1 --dir /dumps
```

### Restore a logical dump

Tables that already exist are skipped unless `--drop` is given (drop and recreate same-named tables).

```bash
mcli --http-addr 127.0.0.1:8300 restore-logical --label daily-1 --dir /dumps
mcli --http-addr 127.0.0.1:8300 restore-logical --label daily-1 --dir /dumps --drop
```

## Argument constraints

- `app-detail`: `--proc` requires `--module`.
- `partition-route`: `--key` and `--start/--end` are mutually exclusive; provide one or the other.
- `command` / `put` / `get` / `range` / `invoke` / `app-invoke`: exactly one of `--json` or `--json-file` is required.
- `repl promote`: `--fencing-cmd` and `--yes` are mutually exclusive; one of them is required.
- `pitr`: `--lsn`, `--time`, and `--xid` are mutually exclusive; exactly one is required.

## Output

- Default output is pretty-printed JSON.
- Add `--compact` for compact JSON.
- On failure the command exits non-zero and prints the error to stderr.
