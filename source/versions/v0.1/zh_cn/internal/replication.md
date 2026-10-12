# 复制架构

复制由 `repl` 与 `mudu_raft` 实现，`server` 和 `mudu_runtime` 负责服务端集成。各 worker 的 WAL 流传输数据，Raft 对提交向量达成共识，而非重新编码 SQL 操作。已提交向量限定从节点的应用范围与主节点恢复边界。

部署形态、配置、快照、成员变更、观测、故障切换及限制统一见[复制与 Raft 容错](../administration/replication.md)。当前复制要求 Tokio 后端。
