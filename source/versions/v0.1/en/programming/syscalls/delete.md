# delete

Delete a key from the session-scoped key-value store.

## Availability

The KV `delete` syscall is declared in the guest/host WIT boundary and implemented by the kernel host, but `sys_interface` does not yet ship a `mudu_delete` wrapper for in-procedure Rust code. The standalone SDK facade exposes it as:

```rust
pub async fn delete(oid: &UniOid, key: &[u8]) -> Result<UniResponse<()>, ApiError>;
```

Until a wrapper lands, procedure code removes relational rows with `command` (`DELETE` statements).

## Conceptual signature

The wire-level signature in `uni-syscall.wit`:

```text
delete: func(oid: uni-oid, key: list<u8>) -> result<_, uni-error>
```

## Parameters

- `oid` — the session whose key-value store is modified.
- `key` — raw key bytes.

## Behavior

- Removes the key from the session's key-value store. As with `put`, the delete is staged in the current session transaction when one is open, and applies directly otherwise.

## Return value

- `()` on success.

## Errors

- `EntityNotFound`: the `oid` does not identify a live session.

## Transaction semantics

- Participates in the current session transaction; outside a transaction the host applies the delete directly (autocommit).

## Notes

- `delete` is the KV counterpart of `read` / `write` / `range-scan`; it is unrelated to SQL `DELETE` statements executed through `command`.
