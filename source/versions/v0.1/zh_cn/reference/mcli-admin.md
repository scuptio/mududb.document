# mcli 管理接口（HTTP）

本页介绍通过 `mcli` 的 HTTP 管理命令完成应用生命周期管理与分区路由查询，以及复制与备份管理命令和 TCP 数据面命令。

## 前置条件

- `mudud` 服务端正在运行。
- HTTP 管理端口可访问（默认 `127.0.0.1:8300`）。

大多数管理命令通过 `--http-addr <host:port>` 指定目标服务器。

## 命令一览

### 列出已安装应用

```bash
mcli --http-addr 127.0.0.1:8300 app-list
```

### 安装应用包

```bash
mcli --http-addr 127.0.0.1:8300 app-install --mpk target/wasm32-wasip2/release/wallet.mpk
```

### 查看应用过程列表

```bash
mcli --http-addr 127.0.0.1:8300 app-detail --app wallet
```

### 查看单个过程详情

`--proc` 必须与 `--module` 一起使用。

```bash
mcli --http-addr 127.0.0.1:8300 app-detail --app wallet --module wallet --proc create_user
```

### 卸载应用

```bash
mcli --http-addr 127.0.0.1:8300 app-uninstall --app wallet
```

### 查看服务拓扑

```bash
mcli --http-addr 127.0.0.1:8300 server-topology
```

### 分区路由查询

按精确 key 路由：

```bash
mcli --http-addr 127.0.0.1:8300 partition-route --rule-name user_rule --key user-100
```

按范围路由：

```bash
mcli --http-addr 127.0.0.1:8300 partition-route --rule-name user_rule --start 100 --end 200
```

多列 key 示例：

```bash
mcli --http-addr 127.0.0.1:8300 partition-route --rule-name order_rule --key tenant-1,order-100
```

## 数据面命令（TCP）

这些命令连接 TCP 协议端点（`--addr`，默认 `127.0.0.1:9527`）。请求体可以用内联 JSON（`--json`）或 JSON 文件（`--json-file`；用 `-` 从 stdin 读取）提供。

SQL 示例假定已安装含 `users(id)` 表的 `demo` 应用。原始 KV 请求必须提供有效会话 OID：`{ "h": 高64位, "l": 低64位 }`；原始 `invoke` 必须提供 `session_id`（u64 数字或十进制 u128 字符串）及编码后的过程参数。下列 ID 与负载是占位示例，不会创建会话。通常应使用 `app-invoke` 自动建立会话并按描述符转换参数。

### 执行 SQL 查询或命令

```bash
mcli --addr 127.0.0.1:9527 command --json '{"app_name":"demo","sql":"SELECT id FROM users"}'
```

### 写入键值项

```bash
mcli --addr 127.0.0.1:9527 put --json '{"oid":{"h":0,"l":1},"key":"k","value":"v"}'
```

### 读取键

```bash
mcli --addr 127.0.0.1:9527 get --json '{"oid":{"h":0,"l":1},"key":"k"}'
```

### 扫描键范围

```bash
mcli --addr 127.0.0.1:9527 range --json '{"oid":{"h":0,"l":1},"start_key":"a","end_key":"z"}'
```

### 以原始请求调用过程

`invoke` 通过 TCP 发送原始过程请求。带类型的变体是 `app-invoke`（见[客户端](../administration/client.md)），它先通过 HTTP 获取过程描述符，再通过 TCP 调用。

```bash
mcli --addr 127.0.0.1:9527 invoke --json '{"session_id":"1","procedure_name":"app/mod/proc","procedure_parameters":{}}'
```

## 复制管理

Raft 成员管理与主从故障切换命令。它们作用于目标节点的 HTTP 管理端点。

### 向 raft 组添加节点

节点默认以 voter 加入；添加 `--learner` 以 learner 加入（只复制不投票；之后用 `promote-learner` 提升）。

```bash
mcli --http-addr 127.0.0.1:8300 repl add-node --node-id 4 --host 10.0.0.4 --port 19530
mcli --http-addr 127.0.0.1:8300 repl add-node --node-id 4 --host 10.0.0.4 --port 19530 --learner
```

### 从 raft 组移除节点

```bash
mcli --http-addr 127.0.0.1:8300 repl remove-node --node-id 4
```

### 将 learner 提升为 voter

```bash
mcli --http-addr 127.0.0.1:8300 repl promote-learner --node-id 4
```

### 将本节点提升为流复制 primary

提升要求提供 fencing 证明：`--fencing-cmd <命令>` 会先在本地执行（经 `sh -c`），失败则中止提升且不接触服务器；或者用 `--yes` 显式确认旧 primary 已被 fence。两个选项互斥。

```bash
mcli --http-addr 127.0.0.1:8301 repl promote --fencing-cmd ./fence-old-primary.sh
mcli --http-addr 127.0.0.1:8301 repl promote --yes
```

### 跟随新的流复制 primary

`--primary` 是新 primary 的复制地址（`host:port`）。

```bash
mcli --http-addr 127.0.0.1:8300 repl follow --primary 10.0.0.2:19527 --primary-node-id 2
```

### 查看复制状态

默认输出 JSON；`--table` 渲染为分节表格。

```bash
mcli --http-addr 127.0.0.1:8300 repl status
mcli --http-addr 127.0.0.1:8300 repl status --table
```

## 备份与恢复

基础备份与逻辑导出通过 HTTP 管理端点在运行中的服务器上执行；`restore` 与 `pitr` 为离线命令，不接触服务器。`[backup]` 配置与磁盘布局见[备份与恢复](../administration/backup.md)。

### 执行基础备份

要求服务器配置 `[backup] backup_dir`。

```bash
mcli --http-addr 127.0.0.1:8300 backup create --label daily-1
```

### 列出基础备份

```bash
mcli --http-addr 127.0.0.1:8300 backup list
```

### 校验基础备份

```bash
mcli --http-addr 127.0.0.1:8300 backup verify --label daily-1
```

### 删除基础备份

```bash
mcli --http-addr 127.0.0.1:8300 backup delete --label daily-1
```

### 恢复基础备份（离线）

将基础备份加归档 WAL 重放到 `--target-dir`，直至归档末尾。

```bash
mcli restore --label daily-1 --backup-dir /backups --archive-dir /wal-archive --target-dir /restored
```

### 时间点恢复（离线）

与 `restore` 类似，但在目标点停止。`--lsn`、`--time`、`--xid` 必须且只能提供一个。

```bash
mcli pitr --label daily-1 --backup-dir /backups --archive-dir /wal-archive --target-dir /recovered --time 1700000000000
```

### 执行逻辑导出

在服务器上写出 `<dir>/<label>.zip`（`--dir` 是服务器端路径）。

```bash
mcli --http-addr 127.0.0.1:8300 dump --label daily-1 --dir /dumps
```

### 执行逻辑导入

默认跳过已存在的同名表；传入 `--drop` 则删除并重建同名表。

```bash
mcli --http-addr 127.0.0.1:8300 restore-logical --label daily-1 --dir /dumps
mcli --http-addr 127.0.0.1:8300 restore-logical --label daily-1 --dir /dumps --drop
```

## 参数约束

- `app-detail`：`--proc` 必须与 `--module` 一起使用。
- `partition-route`：`--key` 与 `--start/--end` 二选一，不可同时使用。
- `command` / `put` / `get` / `range` / `invoke` / `app-invoke`：`--json` 与 `--json-file` 必须且只能提供一个。
- `repl promote`：`--fencing-cmd` 与 `--yes` 互斥；必须提供其一。
- `pitr`：`--lsn`、`--time`、`--xid` 互斥；必须且只能提供一个。

## 返回结果

- 默认输出格式化 JSON。
- 添加 `--compact` 输出紧凑 JSON。
- 出错时命令返回非 0，并在 stderr 输出错误原因。
