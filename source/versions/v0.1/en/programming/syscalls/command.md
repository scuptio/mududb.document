# command

Execute a single SQL `INSERT`, `UPDATE`, or `DELETE` statement.

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

## Conceptual signature

```text
command(sql: string, params?: [Value]) -> affected_rows: u64
```

## Parameters

- `sql` — a single SQL statement using `?` as parameter placeholders.
- `params` — values bound to the placeholders.

## Return value

- `u64` — the number of rows affected by the statement.

## Errors

Invalid SQL can return `Parse`; invalid session/schema/type and transaction conflicts return the host error. Inspect the actual `MuduError` code and message; the API does not define separate `SyntaxError` or `PartialFailure` result types.

## Transaction semantics

- The command participates in the current session transaction.
- Outside a transaction, behavior is host-defined (autocommit or reject).

## Example

```rust
let affected = mudu_command(
    oid,
    sql_stmt!("UPDATE accounts SET balance = balance - ? WHERE id = ?"),
    sql_params!(&(amount, from_id)),
)?;
assert_eq!(affected, 1);
```

## Notes

- Use `command` for DML statements. For `SELECT`, use `query`.
- For multiple statements in one call, consider `batch`.
- `get`, `put`, and `range` address raw byte-key KV data. For typed relational results use `query`; see [Relation Syscalls](relation.md) for direct row access.
