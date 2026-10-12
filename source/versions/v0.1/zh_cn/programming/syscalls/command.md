# command

执行单条 SQL `INSERT`、`UPDATE` 或 `DELETE` 语句。

## Rust API

```rust
// sync_api
pub fn mudu_command(
    oid: OID,
    sql: &dyn SQLStmt,
    params: &dyn SQLParams,
) -> RS<u64> {
    /* ... */
}

// async_api
pub async fn mudu_command(
    oid: OID,
    sql: &dyn SQLStmt,
    params: &dyn SQLParams,
) -> RS<u64> {
    /* ... */
}
```

## 概念签名

```text
command(sql: string, params?: [Value]) -> affected_rows: u64
```

## 参数

- `sql` —— 单条 SQL 语句，使用 `?` 作为参数占位符。
- `params` —— 绑定到占位符的值。

## 返回值

- `u64` —— 语句影响的行数。

## 错误

无效 SQL 可返回 `Parse`；无效会话、schema、类型与事务冲突通过宿主错误返回。请检查实际 `MuduError` 的 code 和 message；该 API 没有单独的 `SyntaxError` 或 `PartialFailure` 结果类型。

## 事务语义

- 该命令参与当前会话事务。
- 事务外行为由宿主定义（自动提交或拒绝）。

## 示例

```rust
let affected = mudu_command(
    oid,
    sql_stmt!("UPDATE accounts SET balance = balance - ? WHERE id = ?"),
    sql_params!(&(amount, from_id)),
)?;
assert_eq!(affected, 1);
```

## 注意

- `command` 用于 DML 语句；`SELECT` 请使用 `query`。
- 如需一次调用执行多条语句，可考虑 `batch`。
- `get`、`put`、`range` 访问原始字节键 KV 数据。类型化关系结果使用 `query`，直接行访问见 [Relation 系统调用](relation.md)。
