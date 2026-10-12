# get

Read a single record from the current system session's key-value store by key. The underlying syscall name is `mudu_get`.

## Rust API

```rust
// sync_api
pub fn mudu_get(session_id: OID, key: &[u8]) -> RS<Option<Vec<u8>>> {
    /* ... */
}

// async_api
pub async fn mudu_get(session_id: OID, key: &[u8]) -> RS<Option<Vec<u8>>> {
    /* ... */
}
```

## Parameters

- `session_id` — session ID returned by `mudu_open`.
- `key` — raw key bytes.

## Behavior

- Performs a point lookup in the session-level key-value store, returning the raw value bytes on hit or `None` on miss.

## Return value

- `Option<Vec<u8>>` — the value on hit; `None` when no match.

## Errors

- `NotFound` is not an error: `get` returns `None` on a miss.

## Example

```rust
let value = mudu_get(oid, b"accounts:42")?;
if let Some(bytes) = value {
    // decode bytes using the application-defined KV encoding
}
```
