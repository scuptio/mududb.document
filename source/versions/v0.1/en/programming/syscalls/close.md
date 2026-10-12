# close

Close the previously opened session, releasing session-scoped resources.

## Rust API

```rust
// sync_api
pub fn mudu_close(session_id: OID) -> RS<()> {
    /* ... */
}

// async_api
pub async fn mudu_close(session_id: OID) -> RS<()> {
    /* ... */
}
```

## Conceptual signature

```text
close(session_id: OID) -> ()
```

## Parameters

- `session_id` — the `OID` returned by a previous `open` call.

## Return value

- `()` on success.

## Errors

- `EntityNotFound`: the provided `OID` does not identify an open session.

## Transaction semantics

- If a transaction is open when `close` is called, behavior depends on the host lifecycle (automatic commit or rollback).
- In MuduDB procedures, the kernel typically manages the transaction boundary around the procedure call.

## Example

```rust
let oid = mudu_open()?;
// ... use oid ...
mudu_close(oid)?;
```

## Notes

- Procedures must close sessions they explicitly open to avoid resource leaks.
- Do not use a closed `OID` after `close`; doing so yields `EntityNotFound`.
