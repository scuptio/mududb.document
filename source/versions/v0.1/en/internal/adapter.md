# Adapters

The adapter layer connects MuduDB client code to different backend capabilities, including SQLite, PostgreSQL, MySQL, and the MuduDB kernel itself. It lets the same procedure code run interactively against a standalone database during development and be deployed against `mudud` in production.

## Supported backends

| Backend | Use case | Connection hint |
|---------|----------|-----------------|
| MuduDB kernel | Production / full Mudu Procedure runtime. | `mudud://host:port/...` |
| SQLite | Local development and tests. | `sqlite://path/to/db.sqlite` |
| PostgreSQL | Compatibility testing against Postgres. | `postgres://user:pass@host/db` |
| MySQL | Compatibility testing against MySQL. | `mysql://user:pass@host/db` |

The exact URL format depends on the adapter implementation in use. Check the current `mudu_adapter` crate for the supported connection strings in this release.

## Connection configuration

Adapters are configured through the `MUDU_CONNECTION` environment variable, which holds the connection string. Common parameters include:

- `http_addr` — HTTP management endpoint used to discover topology when connecting to `mudud`.
- `async_session_loop` — enable or disable the background async manager thread.
- `app` — application name, given as the path segment of the `mudud://` URL (defaults to `default`).

Example connection string:

```text
mudud://127.0.0.1:9527/ycsb?http_addr=127.0.0.1:8300
```

## Async session loop

When `async_session_loop=true`, the adapter forwards session management and I/O commands to a dedicated background manager thread running an async runtime. The manager keeps long-lived `AsyncClient` connections and serializes work through a channel-based command queue.

### Behavioral differences

- **Synchronous path** (`async_session_loop=false`): each `mudu_open` creates a `SyncClient` and calls are performed blocking on the caller thread.
- **Asynchronous path** (`async_session_loop=true`): adapter methods enqueue commands (`Open`/`Close`/`Get`/`Put`/`Range`/`Query`/`Command`/`Batch`) to the manager and wait for a response over a small sync channel, reducing blocking on the calling thread and improving concurrency.

### When to use async loop

Prefer `async_session_loop=true` when:

- connecting to the `IOUring` or `Tokio` backend,
- running many concurrent sessions from a single client,
- blocking the caller thread is expensive.

Use `false` for simple scripts, tests, or when debugging step-by-step.

## Known differences

- The MuduDB kernel supports Mudu Procedures and `.mpk` packages; SQLite/PostgreSQL/MySQL adapters do not.
- Parameterized `batch` is not supported in the current implementation: every backend returns `NotImplemented` when parameters are supplied.
- Partition routing and topology discovery are kernel-specific features.

See [Partitioning](partitioning.md) for routing details and [Mudu Procedure](../programming/mudu_procedure.md) for the procedure programming model.

The async session manager may improve connection reuse and concurrency, but a synchronous facade call still waits for its reply. It does not make the caller nonblocking. For explicit native runtime selection and differences from hosted MPK execution, see [SDKs and Execution Modes](../programming/sdk.md).
