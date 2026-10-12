# MuduDB 服务端配置契约 v1

## 范围

本文档规定 `mudud.cfg` 服务端配置文件格式。该文件控制服务端监听端口、执行模式、运行时路径与 io_uring 行为。

## 版本历史

| 版本 | 日期 | 摘要 |
|------|------|------|
| 1 | 2026-6-25 | 初始 TOML 配置。无显式 `version` 字段；兼容性通过 serde 默认值与别名隐式保证。 |

## 文件位置

显式指定 `--cfg` / `-c` 时只读取该路径；文件缺失或无效会报错，不会回退查找其他配置。未指定时依次查找 `./mudud.cfg`、`~/.mududb/mudud.cfg`；都不存在则返回 `NotFound`。相对存储路径以服务端进程的工作目录为基准。

## 字段与校验

全部根字段及 `[invocation_limits]`、`[replication]`、`[backup]` 表的默认值和校验规则见[配置参考](../../administration/configuration.md)。

## 兼容性

配置文件没有序列化的 `version` 字段。仅标记了 Serde 默认值的字段可以省略，必填字段缺失仍会解码失败。当前根级未知字段会被忽略，而 `[invocation_limits]` 拒绝未知字段。任意添加 `version = 2` 不会选择新解析器，也不能保证旧版本拒绝加载，回滚前需核对目标二进制的兼容性。

`server_mode` 不区分大小写，接受 `Legacy`、`IOUring`（也接受 `io_uring`）和 `Tokio`；路由模式接受 `ConnectionId` / `connection_id`、`PlayerId` / `player_id`、`RemoteHash` / `remote_hash`。`component_target` 接受 `p2` 或 `p3`。

`db_path` 和 `page_size` 属于持久化配置。切换路径即选择另一数据库；变更已有数据的页大小需要迁移或重新初始化。多数配置变更需重启。`tcp_multi_port` 的可变性分类本身不代表已有配置热重载命令。

## 实现依据

- `crates/db-kernel/mudu_runtime/src/backend/mudud_cfg/{mod,io,replication}.rs`
- `crates/db-kernel/runtime_wasm/src/service/invocation_limits.rs`
