# range

Scan key-value pairs in the current system session within a key range. The underlying syscall name is `mudu_range`.

## Rust API

```rust
// sync_api
pub fn mudu_range(
    session_id: OID,
    start_key: &[u8],
    end_key: &[u8],
) -> RS<Vec<(Vec<u8>, Vec<u8>)>> {
    /* ... */
}

// async_api
pub async fn mudu_range(
    session_id: OID,
    start_key: &[u8],
    end_key: &[u8],
) -> RS<Vec<(Vec<u8>, Vec<u8>)>> {
    /* ... */
}
```

## Parameters

- `session_id` — session ID returned by `mudu_open`.
- `start_key` / `end_key` — half-open range `[start_key, end_key)` (start included, end excluded); an empty `end_key` is unbounded.

## Return value

- `Vec<(Vec<u8>, Vec<u8>)>` — a list of `(key, value)` pairs matching the range.

## Example

```rust
let rows = mudu_range(
    oid,
    b"accounts:100",
    b"accounts:200",
)?;
for (k, v) in rows {
    // decode k/v using the application-defined KV encoding
}
```

## Notes

- This API materializes the matching pairs and has no limit/cursor parameter; bound ranges explicitly to control memory use.
