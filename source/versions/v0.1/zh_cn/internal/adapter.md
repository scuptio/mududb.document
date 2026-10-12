# 适配器

适配器层将 MuduDB 客户端代码连接到不同的后端能力，包括 SQLite、PostgreSQL、MySQL 以及 MuduDB 内核本身。它使同一份过程代码可以在开发阶段针对独立数据库交互式运行，也可以在生产环境部署到 `mudud`。

## 支持的后端

| 后端 | 使用场景 | 连接提示 |
|------|----------|----------|
| MuduDB 内核 | 生产环境 / 完整 Mudu 过程运行时。 | `mudud://host:port/...` |
| SQLite | 本地开发与测试。 | `sqlite://path/to/db.sqlite` |
| PostgreSQL | 与 Postgres 的兼容性测试。 | `postgres://user:pass@host/db` |
| MySQL | 与 MySQL 的兼容性测试。 | `mysql://user:pass@host/db` |

具体 URL 格式取决于当前使用的适配器实现。本版本支持的连接字符串请查看 `mudu_adapter` crate。

## 连接配置

适配器通过 `MUDU_CONNECTION` 环境变量配置，该变量保存连接字符串。常见参数包括：

- `http_addr` —— 连接 `mudud` 时用于发现拓扑的 HTTP 管理端点。
- `async_session_loop` —— 是否启用后台异步管理线程。
- `app` —— 应用名，由 `mudud://` URL 的路径段给出（默认为 `default`）。

示例连接字符串：

```text
mudud://127.0.0.1:9527/ycsb?http_addr=127.0.0.1:8300
```

## 异步会话循环

当 `async_session_loop=true` 时，适配器将会话管理与 I/O 命令转发给运行异步运行时的专用后台管理线程。该管理线程保持长期存活的 `AsyncClient` 连接，并通过基于通道的命令队列序列化工作。

### 行为差异

- **同步路径**（`async_session_loop=false`）：每次 `mudu_open` 创建一个 `SyncClient`，调用在调用线程上以阻塞方式执行。
- **异步路径**（`async_session_loop=true`）：适配器方法将命令（`Open`/`Close`/`Get`/`Put`/`Range`/`Query`/`Command`/`Batch`）加入管理线程队列，并通过小型同步通道等待响应，从而减少调用线程上的阻塞并提高并发性。

### 何时使用异步循环

在以下场景优先使用 `async_session_loop=true`：

- 连接 `IOUring` 或 `Tokio` 后端；
- 单个客户端运行大量并发会话；
- 阻塞调用线程代价较高。

在简单脚本、测试或逐步调试时可使用 `false`。

## 已知差异

- MuduDB 内核支持 Mudu 过程与 `.mpk` 包；SQLite/PostgreSQL/MySQL 适配器不支持。
- 当前实现不支持参数化 `batch`：所有后端在提供参数时都返回 `NotImplemented`。
- 分区路由与拓扑发现是内核专属特性。

路由细节请参见[分区](partitioning.md)，过程编程模型请参见 [Mudu 过程](../programming/mudu_procedure.md)。
