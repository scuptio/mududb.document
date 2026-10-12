# batch

Execute multiple SQL statements in one request. The workspace Rust API is:

```rust
pub fn mudu_batch(oid: OID, sql: &dyn SQLStmt, params: &dyn SQLParams) -> RS<u64>;
// async_api: same arguments and result, with async fn.
```

`sql` contains SQL text, not an array of primitive operations. `params` must be empty; parameterized batches return `NotImplemented`. The return value is an affected-row count, not a per-statement result set. There is no `atomic` option or `BatchResult` API.

The batch runs in the current session/transaction context. Its atomicity follows that context and backend; grouping statements in one request does not independently guarantee atomic commit. For user-supplied values, prefer separate parameterized `mudu_command` calls within a procedure rather than interpolating values into batch text.

```rust
mudu_batch(
    oid,
    sql_stmt!("INSERT INTO accounts (id, balance) VALUES (3, 3000); UPDATE accounts SET balance = balance - 500 WHERE id = 2;"),
    sql_params!(&()),
)?;
```
