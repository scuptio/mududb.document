# batch

在一个请求中执行多条 SQL 语句。仓库内 Rust API 为：

```rust
pub fn mudu_batch(oid: OID, sql: &dyn SQLStmt, params: &dyn SQLParams) -> RS<u64>;
// async_api: same arguments and result, with async fn.
```

`sql` 是 SQL 文本，不是原语操作数组。`params` 必须为空；带参数的 batch 返回 `NotImplemented`。返回值为受影响行数，不是逐语句结果集。没有 `atomic` 选项或 `BatchResult` API。

batch 使用当前会话/事务上下文；原子性由上下文和后端决定，将多条语句放在一个请求中并不能独立保证原子提交。对于用户输入，应在过程内分别调用参数化的 `mudu_command`，不要将值拼接到 batch SQL 中。

```rust
mudu_batch(
    oid,
    sql_stmt!("INSERT INTO accounts (id, balance) VALUES (3, 3000); UPDATE accounts SET balance = balance - 500 WHERE id = 2;"),
    sql_params!(&()),
)?;
```
