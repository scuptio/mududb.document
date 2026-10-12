# range

在当前系统会话中，按键范围扫描键值对。底层系统调用名为 `mudu_range`。

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

## 参数

- `session_id` —— `mudu_open` 返回的会话 ID。
- `start_key` / `end_key` —— 左闭右开区间 `[start_key, end_key)`，包含起点、不包含终点；空 `end_key` 表示无上界。

## 返回值

- `Vec<(Vec<u8>, Vec<u8>)>` —— 匹配范围的 `(key, value)` 对列表。

## 示例

```rust
let rows = mudu_range(
    oid,
    b"accounts:100",
    b"accounts:200",
)?;
for (k, v) in rows {
    // 按应用定义的 KV 编码解码 k/v
}
```

## 注意

- 该 API 一次返回所有匹配对，没有 limit/cursor 参数；应显式缩小扫描区间以控制内存用量。
