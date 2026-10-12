# 入门

本章面向首次使用 MuduDB 的用户。

```{toctree}
:maxdepth: 2

install
first-session
```

## 安装路径

本工作区请使用[源码构建流程](install.md)。计划中的 `mudup` 发布版安装器尚未实现。语言分发包与本地应用见 [SDK 与执行模式](../../programming/sdk.md)。

## 连接字符串 (MUDU_CONNECTION)

适配器支持 `mudud://` 连接 URI 格式，由 `MUDU_CONNECTION` 环境变量使用。格式如下：

```
mudud://<addr>[/<app_name>][?query]
```

- `addr` — mudud TCP 端点的 host:port（例如 `127.0.0.1:9527`）。
- `app_name` — 可选的应用名称；如果省略，默认为 `default`。
- `query` — 可选的 key=value 对，以 `&` 分隔。

可识别的查询键：

- `http_addr` / `http` / `admin_addr` — 用于获取后端拓扑的管理 HTTP 地址（默认 `127.0.0.1:8300`）。
- `async_session_loop` / `async_sessions` / `async` — 布尔标志（`1`/`true`/`yes`/`on`），用于启用异步会话循环。

示例：

```
export MUDU_CONNECTION="mudud://127.0.0.1:9527/ycsb?http_addr=127.0.0.1:8300"
```
