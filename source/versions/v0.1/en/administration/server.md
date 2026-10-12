# Server Management

`mudud` hosts the database and installed MPK applications. Listeners depend on `server_mode`: the default `Tokio` and Linux `IOUring` backends expose kernel TCP and HTTP management; `Legacy` registers HTTP and PostgreSQL wire services. The kernel modes do not start a PostgreSQL listener merely because `pg_listen_port` is configured.

## Prerequisites

- `mudud` and `mcli` are installed and in `PATH`.
- A configuration file exists at `./mudud.cfg`, `~/.mududb/mudud.cfg`, or a path passed with `--cfg`.
- On Linux, `IOUring` requires kernel and process permissions that allow io_uring; `liburing-dev` is a build dependency.
- The process should be started with a high file-descriptor limit (`ulimit -n 65535` is a practical baseline).

## Starting the server

### Default start

```bash
ulimit -n 65535
mudud
```

If `./mudud.cfg` does not exist, `mudud` checks `~/.mududb/mudud.cfg`. Create a default config with:

```bash
mudud init-cfg
```

### Start with a custom config

```bash
ulimit -n 65535
mudud serve --cfg ./config/mudud.cfg
```

### Listeners

| Mode | TCP data plane | HTTP management | PostgreSQL wire |
|------|----------------|-----------------|-----------------|
| `Tokio` / `IOUring` | `tcp_listen_port` (9527) | `http_listen_port` (8300) | Not started |
| `Legacy` | No kernel TCP listener | `http_listen_port` (8300) | `pg_listen_port` (5432) |

With multiple kernel workers, the backend enables consecutive worker TCP ports; `server-topology` reports the actual endpoints. Bind addresses use `listen_ip`, default `127.0.0.1`.

## Stopping the server

Send `SIGINT` (`Ctrl+C`) or `SIGTERM`. `mudud` performs a graceful shutdown, waiting for in-flight work to finish before exiting.

## Verifying the server

Use `mcli` to check the HTTP management endpoint:

```bash
mcli --http-addr 127.0.0.1:8300 app-list
mcli --http-addr 127.0.0.1:8300 server-topology
```

If the commands return JSON output, the server is running and reachable.

## Server modes

| Mode | When to use |
|------|-------------|
| `Legacy` | Legacy HTTP/PostgreSQL serving path; no kernel TCP endpoint. |
| `IOUring` | Linux io_uring kernel backend; replication is unavailable. |
| `Tokio` | Default kernel backend; required for replication. |

## Common issues

- **Address already in use**: another process is using one of the configured ports. Change the conflicting port in `mudud.cfg`.
- **Permission denied**: you may not have permission to bind to the configured `listen_ip` or port. Use `127.0.0.1` and ports above `1024` for local development.
- **Too many open files**: raise the file descriptor limit with `ulimit -n 65535` before starting.
- **io_uring not available**: if the kernel or process environment does not allow io_uring, switch `server_mode` to `Tokio`.

See the [Configuration Reference](configuration.md) for all available options.
