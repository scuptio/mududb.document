# Client

`mcli` is the MuduDB command-line client. It supports interactive SQL, HTTP administration, and procedure invocation.

## Connection options

- `--addr <host:port>` — TCP protocol endpoint for SQL shell and procedure invocation (default `127.0.0.1:9527`).
- `--http-addr <host:port>` — HTTP management endpoint for admin commands (default `127.0.0.1:8300`).

## Interactive SQL shell

```bash
mcli --addr 127.0.0.1:9527 shell --app demo
```

Inside the shell you can run SQL statements, meta commands, and exit with `\q`.

## Common admin commands

List installed applications:

```bash
mcli --http-addr 127.0.0.1:8300 app-list
```

Show server topology:

```bash
mcli --http-addr 127.0.0.1:8300 server-topology
```

Resolve a partition route:

```bash
mcli --http-addr 127.0.0.1:8300 partition-route --rule-name user_rule --key user-100
```

## Invoke a procedure

```bash
mcli --addr 127.0.0.1:9527 --http-addr 127.0.0.1:8300 app-invoke \
  --app wallet \
  --module wallet \
  --proc transfer_funds \
  --json '{"from_user_id": 1001, "to_user_id": 1002, "amount": 1200}'
```

## Standalone adapter connection

Programs that run procedures through the standalone adapter (`mudu_adapter`) — rather than through `mcli` — read the `MUDU_CONNECTION` environment variable to pick a backend. When the variable is unset, the adapter defaults to `sqlite:///tmp/mududb.db`.

| Scheme | Backend | Example |
|--------|---------|---------|
| `sqlite://<path>` | SQLite file | `sqlite:///tmp/mududb.db` |
| `postgres://…`, `postgresql://…` | PostgreSQL | `postgres://postgres:postgres@127.0.0.1:5432/tpcc` |
| `mysql://…` | MySQL | `mysql://root:root@127.0.0.1:3306/tpcc` |
| `mudud://<addr>[/<app>][?query]` | MuduDB server | `mudud://127.0.0.1:9527/ycsb?http_addr=127.0.0.1:8300` |

A `sqlite:` prefix or a bare path is also treated as a SQLite file path. For the `mudud://` form, the app name defaults to `default` and the `http_addr` query parameter defaults to `127.0.0.1:8300`; see [Adapter configuration](../internal/adapter.md) for the query parameters.

## Output

- Query results are displayed as a table by default in TTY sessions.
- Management commands output pretty JSON by default; add `--compact` for compact JSON.
- On error, `mcli` exits non-zero and prints the error to stderr.

See [mcli Management Interface (HTTP)](../reference/mcli-admin.md) for the full command reference.
