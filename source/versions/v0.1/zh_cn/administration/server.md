# 服务端管理

`mudud` 托管数据库与已安装的 MPK 应用。监听端点取决于 `server_mode`：默认 `Tokio` 和 Linux `IOUring` 后端提供内核 TCP 与 HTTP 管理服务；`Legacy` 注册 HTTP 与 PostgreSQL wire 服务。内核模式不会仅因配置了 `pg_listen_port` 就启动 PostgreSQL 监听。

## 前置条件

- `mudud` 与 `mcli` 已安装并位于 `PATH`。
- 配置文件存在于 `./mudud.cfg`、`~/.mududb/mudud.cfg`，或通过 `--cfg` 指定的路径。
- Linux 的 `IOUring` 模式要求内核及进程权限允许 io_uring；`liburing-dev` 是构建依赖。
- 建议以较高的文件描述符限制启动（`ulimit -n 65535` 是一个实用基线）。

## 启动服务端

### 默认启动

```bash
ulimit -n 65535
mudud
```

若当前目录不存在 `./mudud.cfg`，`mudud` 会检查 `~/.mududb/mudud.cfg`。可用以下命令生成默认配置：

```bash
mudud init-cfg
```

### 使用自定义配置启动

```bash
ulimit -n 65535
mudud serve --cfg ./config/mudud.cfg
```

### 监听端点

| 模式 | TCP 数据面 | HTTP 管理 | PostgreSQL wire |
|------|------------|-----------|-----------------|
| `Tokio` / `IOUring` | `tcp_listen_port`（9527） | `http_listen_port`（8300） | 不启动 |
| `Legacy` | 不启动内核 TCP 监听 | `http_listen_port`（8300） | `pg_listen_port`（5432） |

内核后端使用多个 worker 时会启用连续的 worker TCP 端口；实际端点可由 `server-topology` 查询。绑定地址使用 `listen_ip`，默认为 `127.0.0.1`。

## 停止服务端

向进程发送 `SIGINT`（`Ctrl+C`）或 `SIGTERM`。`mudud` 会执行优雅关闭，等待在途工作完成后再退出。

## 验证服务端

使用 `mcli` 检查 HTTP 管理端点：

```bash
mcli --http-addr 127.0.0.1:8300 app-list
mcli --http-addr 127.0.0.1:8300 server-topology
```

若命令返回 JSON 输出，说明服务端正在运行且可访问。

## 服务端模式

| 模式 | 适用场景 |
|------|----------|
| `Legacy` | 旧版 HTTP/PostgreSQL 服务路径；无内核 TCP 端点。 |
| `IOUring` | Linux io_uring 内核后端；不支持复制。 |
| `Tokio` | 默认内核后端；复制必须使用此模式。 |

## 常见问题

- **地址已被占用**：其他进程占用了某个配置端口。在 `mudud.cfg` 中修改冲突端口。
- **权限被拒绝**：可能没有权限绑定到配置的 `listen_ip` 或端口。本地开发建议使用 `127.0.0.1` 与 1024 以上端口。
- **打开文件过多**：启动前使用 `ulimit -n 65535` 提高文件描述符限制。
- **io_uring 不可用**：若内核或进程环境不允许 io_uring，将 `server_mode` 切换为 `Tokio`。

所有可用选项请参见[配置参考](configuration.md)。
