# query

Run a SQL `SELECT` statement and return the result set with rows mapped to Rust structs (R2O, relation-to-object mapping).

## Rust API

```rust
// sync_api
pub fn mudu_query<R: Entity>(
    oid: OID,
    sql: &dyn SQLStmt,
    params: &dyn SQLParams,
) -> RS<EntitySet<R>> {
    /* ... */
}

// async_api
pub async fn mudu_query<R: Entity>(
    oid: OID,
    sql: &dyn SQLStmt,
    params: &dyn SQLParams,
) -> RS<EntitySet<R>> {
    /* ... */
}
```

## Conceptual signature

```text
query<R: Entity>(sql: string, params?: [Value]) -> EntitySet<R>
```

## Parameters

- `oid` — the current system session identifier passed into the procedure.
- `sql` — a single `SELECT` statement using `?` as parameter placeholders.
- `params` — the values bound to the placeholders.

## Behavior

- Executes the `SELECT` in the current session and automatically maps each result row to the entity type `R` (R2O mapping); `R` must implement the `Entity` trait (see [Mudu Procedure](../mudu_procedure.md)).
- Row visibility follows the current transaction's isolation rules, as with any other read.

## Return value

- `RS<EntitySet<R>>` — the mapped result set. Call `next_record()` to iterate: it returns `RS<Option<R>>`, yielding `Ok(Some(row))` for each row and `Ok(None)` at the end of the result set.

## Errors

- `Parse`: the SQL text cannot be parsed.
- `EntityNotFound`: the `oid` does not identify a live session.
- `Database`: the storage engine failed to execute the statement.

## Transaction semantics

- The query participates in the current session transaction.
- A procedure invocation runs as one transaction that commits on `Ok` and rolls back on `Err`; see [Mudu Procedure](../mudu_procedure.md).

## Example

```rust
use sys_interface::sync_api::mudu_query;

let wallets = mudu_query::<Wallets>(
    oid,
    sql_stmt!(&Wallets::SQL_GET_BY_PK),
    sql_params!(&(user_id,)),
)?;
if let Some(wallet) = wallets.next_record()? {
    // wallet is a Wallets struct mapped from the selected row
}
```

## Notes

- Use `query` only for `SELECT`. For `INSERT` / `UPDATE` / `DELETE` use `command`; for multiple statements in one call use `batch`.
- Entity modules are normally generated with `mgen entity`, which also emits the `SQL_GET_BY_PK` / `SQL_INSERT` / `SQL_DELETE_BY_PK` text constants used above.
