# Partitioning

MuduDB implements global RANGE partition rules, table bindings and explicit partition-to-worker placement. The table's partition columns must form a primary-key prefix. Bounds are lexicographic key tuples; ranges include the lower bound and exclude the upper bound. Multiple tables can share a rule for colocated access.

## Rule and table

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

## Placement and routing

`CREATE PARTITION PLACEMENT FOR RULE r_orders (PARTITION p0 ON WORKER <worker_id>, PARTITION p1 ON WORKER <worker_id>);` assigns each partition. Substitute actual numeric worker OIDs discovered from topology; a worker ID is not simply its array index. Execute administrative DDL through an admin-capable session.

```bash
mcli --http-addr 127.0.0.1:8300 server-topology
mcli --http-addr 127.0.0.1:8300 partition-route --rule-name r_orders --key 1001,50001
```

`PartitionRouter` resolves the table binding, compares the key with range bounds, and obtains the worker from placement metadata. Range requests can involve more than one partition. The adapter can use the topology to connect directly to a worker and open a worker-bound session.

Hashing a key modulo the worker count is not a substitute for the configured RANGE rule. Topology reports current placement; it is not a promise of automatic worker reassignment or live resharding. Cross-worker writes may be forwarded, but distributed two-phase atomic commit is not implemented.

Implementation: `crates/common/sql_parser/src/ast/stmt_create_partition_rule.rs`, `stmt_create_partition_placement.rs`, and `crates/db-kernel/xengine/src/partition_router.rs`.
