# open

Open a session or logical connection context for a procedure. Sessions group transactional and execution context for subsequent syscalls.

## Rust API

```rust
// sync_api
pub fn mudu_open() -> RS<OID> {
    /* ... */
}

// async_api
pub async fn mudu_open() -> RS<OID> {
    /* ... */
}
```

## Conceptual signature

```text
open() -> OID
```

## Parameters

None.

## Return value

- `OID` — a session identifier used as the first argument to subsequent syscalls such as `get`, `put`, `query`, and `command`.

## Errors

Session creation can fail with a host error, including invocation resource-budget exhaustion. Native calls require an explicitly configured standalone adapter; the plain host-side facade can return `NotImplemented`.

## Transaction semantics

- Opening a session does not implicitly start a durable transaction.
- Transaction boundaries are controlled by the host; the guest interface exposes no dedicated commit/rollback syscall.

## Example

```rust
let oid = mudu_open()?;
let value = mudu_get(oid, b"accounts:42")?;
mudu_close(oid)?;
```

## Notes

- In procedure code the session `OID` is usually passed in as the first argument, so explicit `open`/`close` are most common in interactive tests or standalone adapter code.
- Always pair `open` with `close` to release session-scoped resources.
