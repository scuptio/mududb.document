# batch

Execute multiple operations as a single batched request.

## Rust API

```rust
// sync_api
pub fn mudu_batch(oid: OID, sql: &dyn SQLStmt, params: &dyn SQLParams) -> RS<u64> {
    /* ... */
}

// async_api
pub async fn mudu_batch(oid: OID, sql: &dyn SQLStmt, params: &dyn SQLParams) -> RS<u64> {
    /* ... */
}
```

`mudu_batch` uses the same argument and return contract as `mudu_command`. `batch` executes raw multi-statement SQL text and does not support SQL parameters: the parameter list must be empty, otherwise the runtime returns `NotImplemented`.

## Conceptual signature

```text
batch(ops: [string | primitive-op], options?: { atomic: bool }) -> BatchResult
```

## Parameters

- `ops` — an ordered list of concrete SQL statements or primitive operations. Parameterized SQL strings are not supported.
- `options.atomic` — when true, the runtime attempts to execute the batch as an atomic group (all-or-nothing). Atomic batches may be rejected depending on the storage engine.

## Behavior

- Executes ops in sequence under the current session and transaction context.
- If any op fails and `options.atomic` is true, the runtime attempts to roll back prior ops in the batch; rollback guarantees depend on storage support.
- If parameters are provided with SQL strings, the runtime returns `NotImplemented`.

## Return value

- `BatchResult` contains per-op status entries and a summary (success count, error entries).

## Errors and edge cases

- `NotImplemented` — when a caller supplies SQL parameter placeholders with a non-empty parameter list.
- `ConstraintViolation` — when a mutation violates schema or uniqueness constraints.
- `PartialFailure` — when `atomic=false`, some ops may succeed while others fail; check per-op status.

## Transaction semantics

- Batches participate in the current transaction.
- If no transaction is open, behavior follows host defaults (autocommit or per-op commit).

## Example

```text
-- two concrete statements in a batch
batch([
  "INSERT INTO accounts (id, owner, balance) VALUES (3, 'Eve', 3000)",
  "UPDATE accounts SET balance = balance - 500 WHERE id = 2"
], { atomic: true });
```

## Best practices

- Avoid passing parameter placeholders to batch; expand parameters before batching.
- Use atomic batches sparingly; prefer explicit transactions for complex multi-step updates.
- Prefer smaller batches to reduce lock contention and improve failure isolation.
