# 每核 Worker 模型

内核模式（`Tokio` 与 Linux `IOUring`）使用带本地执行状态和异步 I/O 的 worker 线程。`worker_threads = 0` 选择可用并行度，正数则指定 worker 数量。该配置本身不保证 CPU 绑核。

各 worker 管理会话/事务状态，并为分配给自己的分区执行数据操作。异步任务在等待 I/O 或内部 RPC 时让出执行权。本地操作在所属 worker 执行，分区路由可通过内部消息总线转发远程操作。

多 worker 时，服务端从 `tcp_listen_port` 起启用连续的 worker TCP 端口。客户端通过 `mcli server-topology` 发现 worker ID、端口和分区 placement，再建立指向指定 worker 的会话。`routing_mode` 控制连接路由，不代表 SQL 哈希分区方案。

显式范围规则与 placement 见[分区](partitioning.md)，guest 调度见[可挂起的过程执行](suspendable-execution.md)，节点故障切换见[复制](replication.md)。worker 局部性不代表自动在线分区迁移或分布式原子提交。

实现依据：`crates/db-kernel/server/src/server/serve/`、`crates/db-kernel/worker/src/` 及 `mudu_runtime` 的后端启动代码。
