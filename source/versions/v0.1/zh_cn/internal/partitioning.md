# 分区

MuduDB 实现全局 RANGE 分区规则、表绑定和显式的分区到 worker placement。表的分区列必须构成主键前缀。边界为按字典序比较的键元组，区间左闭右开。多张表可以共享规则以实现同分区访问。

## 规则与表

```sql
CREATE PARTITION RULE r_orders
RANGE (region_id, order_id) (
  PARTITION p0 VALUES FROM (MINVALUE, MINVALUE) TO (1000, MINVALUE),
  PARTITION p1 VALUES FROM (1000, MINVALUE) TO (MAXVALUE, MAXVALUE)
);

CREATE TABLE orders (
  region_id BIGINT,
  order_id BIGINT,
  amount BIGINT,
  PRIMARY KEY(region_id, order_id)
)
PARTITION BY GLOBAL RULE r_orders
REFERENCES (region_id, order_id);
```

## Placement 与路由

通过 `CREATE PARTITION PLACEMENT FOR RULE r_orders (PARTITION p0 ON WORKER <worker_id>, PARTITION p1 ON WORKER <worker_id>);` 分配各分区。请使用 topology 返回的实际数值 worker OID 替换占位符，worker ID 不是简单的数组下标。管理类 DDL 应通过具有管理能力的会话执行。

```bash
mcli --http-addr 127.0.0.1:8300 server-topology
mcli --http-addr 127.0.0.1:8300 partition-route --rule-name r_orders --key 1001,50001
```

`PartitionRouter` 解析表绑定，将键与范围边界比较，再由 placement 元数据确定 worker。范围请求可能覆盖多个分区。适配器可以利用 topology 直连 worker 并建立绑定会话。

对键取哈希再模 worker 数量不能替代已配置的 RANGE 规则。topology 反映当前 placement，不代表自动重新分配 worker 或在线重分片。跨 worker 写入可被转发，但分布式两阶段原子提交尚未实现。

实现依据：`crates/common/sql_parser/src/ast/stmt_create_partition_rule.rs`、`stmt_create_partition_placement.rs` 和 `crates/db-kernel/xengine/src/partition_router.rs`。
