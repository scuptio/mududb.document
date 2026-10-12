# 客户端

`mcli` 是 MuduDB 命令行客户端，支持交互式 SQL、HTTP 管理与过程调用。

## 连接选项

- `--addr <host:port>` —— SQL shell 与过程调用使用的 TCP 协议端点（默认 `127.0.0.1:9527`）。
- `--http-addr <host:port>` —— 管理命令使用的 HTTP 管理端点（默认 `127.0.0.1:8300`）。

## 交互式 SQL shell

```bash
mcli --addr 127.0.0.1:9527 shell --app demo
```

在 shell 中可以执行 SQL 语句、元命令，使用 `\q` 退出。

## 常用管理命令

列出已安装应用：

```bash
mcli --http-addr 127.0.0.1:8300 app-list
```

查看服务拓扑：

```bash
mcli --http-addr 127.0.0.1:8300 server-topology
```

解析分区路由：

```bash
mcli --http-addr 127.0.0.1:8300 partition-route --rule-name user_rule --key user-100
```

## 调用过程

```bash
mcli --addr 127.0.0.1:9527 --http-addr 127.0.0.1:8300 app-invoke \
  --app wallet \
  --module wallet \
  --proc transfer_funds \
  --json '{"from_user_id": 1001, "to_user_id": 1002, "amount": 1200}'
```

## 独立适配器连接（MUDU_CONNECTION）

通过独立适配器（`mudu_adapter`）运行过程的程序——而非通过 `mcli`——读取 `MUDU_CONNECTION` 环境变量来选择后端。该变量未设置时，适配器缺省使用 `sqlite:///tmp/mududb.db`。

| Scheme | 后端 | 示例 |
|--------|------|------|
| `sqlite://<路径>` | SQLite 文件 | `sqlite:///tmp/mududb.db` |
| `postgres://…`、`postgresql://…` | PostgreSQL | `postgres://postgres:postgres@127.0.0.1:5432/tpcc` |
| `mysql://…` | MySQL | `mysql://root:root@127.0.0.1:3306/tpcc` |
| `mudud://<地址>[/<应用>][?query]` | MuduDB 服务器 | `mudud://127.0.0.1:9527/ycsb?http_addr=127.0.0.1:8300` |

`sqlite:` 前缀或裸路径同样按 SQLite 文件路径处理。`mudud://` 形式中应用名默认为 `default`，`http_addr` 查询参数默认为 `127.0.0.1:8300`；查询参数详见[适配器配置](../internal/adapter.md)。

## 输出

- 在 TTY 会话中，查询结果默认以表格形式显示。
- 管理命令默认输出格式化 JSON；添加 `--compact` 可输出紧凑 JSON。
- 出错时 `mcli` 返回非 0 退出码，并将错误输出到 stderr。

完整命令参考请参见 [mcli HTTP 管理接口](../reference/mcli-admin.md)。
