# Relation 系统调用

`relation-get`、`relation-update` 和 `relation-insert` 根据主键访问表中的行，无需发送 SQL 文本；它们与原始字节键 KV 存储不同。WIT 中对应消息类型为 21、22、23。仓库内 Rust facade 的签名如下（`async_api` 提供对应异步函数）：

```rust
pub fn mudu_relation_get(
    session_id: OID, table: &str,
    key: &[(u64, Vec<u8>)], select: &[u64],
) -> RS<Option<Vec<Option<Vec<u8>>>>>;

pub fn mudu_relation_update(
    session_id: OID, table: &str,
    key: &[(u64, Vec<u8>)], values: &[(u64, Vec<u8>)],
    deltas: &[UniRelationDelta],
) -> RS<u64>;

pub fn mudu_relation_insert(
    session_id: OID, table: &str,
    key: &[(u64, Vec<u8>)], values: &[(u64, Vec<u8>)],
) -> RS<()>;
```

列标识为原始表定义中从零开始的列索引，使用 `u64` 传递。值使用对应列的二进制 datum 编码，不是 JSON 或 KV 编码。`key` 指定主键列，`select` 指定结果列顺序。行不存在时 `get` 返回 `None`；行存在时，每列返回可选字节值（`None` 表示 SQL NULL）。`insert` 拒绝重复主键。

`update` 支持赋值与数值增量。`UniRelationDelta::add`、`sub` 对应操作码 0、1。延迟变体（操作码 2/3，以及专用的环绕减法操作码 4）在提交应用阶段执行，调用方必须满足实现规定的可交换性约束，不能将其当作任意 SQL 更新的通用替代。

这些操作参与当前会话事务，不提供跨 worker 的分布式原子提交。独立 Rust SDK 通过 `Mudu::relation_get`、`relation_update`、`relation_insert` 暴露使用 `UniOid` 和 `UniResponse` 的接口；具体执行能力仍取决于所选后端。

实现依据：`crates/common/mudu_binding/wit/uni-syscall.wit`、`crates/common/mudu_binding/src/universal/uni_relation.rs`、`crates/common/sys_interface/src/api_impl/`、`crates/db-kernel/worker/src/worker/relation_ops.rs`。
