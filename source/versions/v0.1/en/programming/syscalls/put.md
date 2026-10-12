# put

Write a key-value pair into the current system session's key-value store. The underlying syscall name is `mudu_put`.

## Rust API

```rust
// sync_api
pub fn mudu_put(session_id: OID, key: &[u8], value: &[u8]) -> RS<()> {
    /* ... */
}

// async_api
pub async fn mudu_put(session_id: OID, key: &[u8], value: &[u8]) -> RS<()> {
    /* ... */
}
```

## Parameters

- `session_id` — session ID returned by `mudu_open`.
- `key` — raw key bytes.
- `value` — raw value bytes.

## Behavior

- Writes or overwrites a key-value pair in the session-level key-value store.

## Return value

- `RS<()>` — empty on success.

## Errors

- Invalid sessions or host/backend failures are returned as `MuduError`; raw KV values are not validated against a table schema.

## Example

```rust
mudu_put(oid, b"accounts:42", b"{\"balance\":1000}")?;
```
