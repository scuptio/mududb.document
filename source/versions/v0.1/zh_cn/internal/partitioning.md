# 分区与工作线程映射

## 概览

分区将应用程序数据和流量划分为逻辑分区（partition_oid），并映射到后端工作线程（worker_oid）。目标是利用局部性：将相关的数据和操作保持在同一工作线程上，以减少跨工作线程通信、降低延迟并提高缓存局部性。

## 设计原理

- 局部性：将相关的键/记录放置在一起，以减少远程调用和状态同步。
- 负载隔离：分区将热点限制在特定工作线程，并通过移动分区简化负载均衡。
- 可预测路由：客户端可以将请求路由到分区所有者，避免不必要的网关跳转。

## 术语

- partition_oid: 分区的逻辑标识符。
- worker_oid: 托管分区的后端工作线程进程标识符。
- sharding port: 工作线程接受特定分片/分区连接的 TCP 端口。
- topology: 管理 HTTP API 数据，将 partition_oid 映射到 worker_oid，并提供工作线程地址。

## 模型与语法

- 分区标识符是拓扑映射中使用的Opaque整数。
- 连接字符串可以嵌入应用层提示；拓扑通过 `MUDU_CONNECTION` 中的 `http_addr` 发现。

示例连接：`mudud://127.0.0.1:9527/ycsb?http_addr=127.0.0.1:8300`

客户端获取拓扑并解析 partition_oid -> worker_oid，然后打开针对 worker OID 的会话。

## 路由工作原理

1. 客户端从管理 HTTP API（http_addr）获取拓扑。
2. 客户端为每个操作计算或获取 partition_oid（例如，通过对键进行哈希或使用应用语义）。
3. 客户端在拓扑缓存中查找 partition_oid -> worker_oid 的映射。
4. 客户端连接到工作线程的分片端口（或带工作线程路由的标准端口），并为该 worker_oid 打开会话（会话打开在配置握手中包含 worker_id）。
5. 路由到工作线程的请求在工作线程本地执行或由工作线程高效代理。



## 拓扑发现

- 使用 HTTP 管理 API `GET /mudu/server/topology` 或 `mcli --http-addr <addr> server-topology` 获取实时拓扑。
- 拓扑 JSON 包含以下字段：
  - `worker_count`
  - `tcp_multi_port` (bool)
  - `tcp_base_listen_port` (int)
  - 每个工作线程的 `worker_index`、`worker_id`、`tcp_listen_port` 和 `partitions` 列表（放置在该工作线程上的分区 id）。

- 要将分区键解析到工作线程：
  1. 调用 `partition-route`，传入规则名称和键元组以获取 `partition_id` 和 `worker_id`。
  2. 从 `server-topology` 结果中查找工作线程的 `tcp_listen_port`。
  3. 连接到 `listen_ip:tcp_listen_port` 并打开包含 `worker_id` 的会话配置。

示例：

```bash
mcli --http-addr 127.0.0.1:8300 partition-route --rule-name r_orders --key 1001,50001
mcli --http-addr 127.0.0.1:8300 server-topology
```

## 后端映射与缓存

- 拓扑在管理 HTTP 端点上是权威的。客户端应在本地缓存映射，并定期或在出错时刷新。
- 缓存失效：当 RPC 错误指示 "NoSuchPartition" 或类似情况时，客户端应使相关条目失效并重新获取拓扑。
- 更新策略：定期的短间隔刷新（秒级）加上出错时的事件驱动刷新。

## 分片端口与工作线程连接

- 工作线程可能暴露绑定到给定分区范围的分片端口，或接受带有 worker_id 握手的连接。
- 如果支持，客户端可以直接连接到工作线程的分片端口以到达该工作线程拥有的分区。
- 如果启用了每个工作线程多端口（tcp_multi_port），服务器会打开每个工作线程的监听器以减少 accept 争用并提供直接路由。

## 获取拓扑并直接连接

- 管理 HTTP API（http_addr）提供 JSON 拓扑映射。示例流程：
  - GET /mudu/server/topology -> 返回工作线程和分区分配列表。
  - 提取目标 worker_oid 的工作线程网络地址和分片端口。
  - 连接到工作线程地址并打开包含 worker_id 的 session_open_config_json 的会话。

## 使用示例

- 基于哈希的分区（客户端侧）：计算 partition_oid = hash(key) % partition_count，然后路由到 topology[partition_oid]。
- 应用分配的分区：应用根据业务逻辑分配 partition_oid（player_id → partition_oid）。

## 故障模式与回退

- 过期映射：在连接/拒绝时，刷新拓扑并使用更新后的映射重试。
- 工作线程故障转移：如果工作线程宕机，拓扑将重新分配分区 —— 客户端必须重新解析并重新连接。
- 保守回退：如果拓扑不可用，客户端可以通过管理/网关工作线程路由请求，该工作线程代理请求，接受更高的延迟。

## 运维关注

- 重新分片：在工作线程之间移动分区需要更新拓扑并排空连接。客户端必须处理重新分片期间的瞬态故障。
- 监控：暴露分区到工作线程的映射和每个分区的指标，以检测热分区。

## 性能与调优

- 分区粒度：较小的分区增加路由分辨率，但增加元数据开销。
- 连接复用：为每个 worker_oid 保持持久会话，以摊销会话打开成本。
- async_session_loop：在高并发场景下使用适配器异步循环，以避免在会话管理上阻塞。

## 故障排除

- 常见错误：NoSuchPartition、ConnectionRefused、NotAuthorized。
- 诊断：获取拓扑、验证分区分配、确认工作线程网络可达性、检查分片端口可用性。


---

分区文档结束（EN）。
