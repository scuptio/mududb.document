# Getting Started

This chapter is for first-time MuduDB users.

```{toctree}
:maxdepth: 2

install
first-session
```

## Installation Paths

````{grid} 1 1 2 2
```{grid-item-card} Release Installation
Use `mudup install` (planned; not yet available) for server deployment, daily use, and quick validation.
```

```{grid-item-card} Source Build
Build from source when developing the kernel, toolchain, API bindings, or example applications.
```
````

## Connection string (MUDU_CONNECTION)

The adapter supports a `mudud://` connection URI format used by the `MUDU_CONNECTION` environment variable. Format:

```
mudud://<addr>[/<app_name>][?query]
```

- `addr` — host:port of the mudud TCP endpoint (e.g. `127.0.0.1:9527`).
- `app_name` — optional application name; defaults to `default` if omitted.
- `query` — optional key=value pairs separated by `&`.

Recognized query keys:

- `http_addr` / `http` / `admin_addr` — management HTTP address used to fetch backend topology (default `127.0.0.1:8300`).
- `async_session_loop` / `async_sessions` / `async` — boolean flag (`1`/`true`/`yes`/`on`) to enable the async session loop.

Example:

```
export MUDU_CONNECTION="mudud://127.0.0.1:9527/ycsb?http_addr=127.0.0.1:8300"
```
