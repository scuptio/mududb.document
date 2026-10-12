# Relation Syscalls

`relation-get`, `relation-update` and `relation-insert` access a table row by its primary key without sending SQL text. They are distinct from the raw byte-key KV store. WIT declares these as message kinds 21, 22 and 23; the workspace Rust facade exposes the signatures below (`async_api` supplies corresponding async functions).

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

Column identifiers are zero-based indexes in the original table definition, carried as `u64`. Values use the column's binary datum encoding, not JSON or the KV encoding. `key` identifies primary-key columns; `select` gives result-column order. `get` returns `None` for a missing row and an optional byte value per selected column for a present row (`None` represents SQL NULL). `insert` rejects duplicate primary keys.

`update` supports assignments and numeric deltas. `UniRelationDelta::add` and `sub` use operation tags 0 and 1. Deferred variants (tags 2/3, and the specialized wrapped decrement tag 4) apply at commit time and require the caller to respect the documented commutativity contract; they are not general replacements for arbitrary SQL updates.

Operations use the current session transaction. They do not add distributed atomic commit across workers. The standalone Rust SDK exposes `Mudu::relation_get`, `relation_update` and `relation_insert` using `UniOid` and `UniResponse`; supported execution paths still depend on the selected backend.

Implementation: `crates/common/mudu_binding/wit/uni-syscall.wit`, `crates/common/mudu_binding/src/universal/uni_relation.rs`, `crates/common/sys_interface/src/api_impl/` and `crates/db-kernel/worker/src/worker/relation_ops.rs`.
