# get

按 key 读取当前系统会话中键值存储的一条记录。底层系统调用名为 `mudu_get`。

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

## 参数

- `session_id` —— `mudu_open` 返回的会话 ID。
- `key` —— 原始键字节。

## 行为

- 在会话级键值存储中做点查，命中返回原始值字节，未命中返回 `None`。

## 返回值

- `Option<Vec<u8>>` —— 命中的值；未命中时为 `None`。

## 错误

- `NotFound` 不是错误：未命中时 `get` 返回 `None`。

## 示例

```rust
let value = mudu_get(oid, b"accounts:42")?;
if let Some(bytes) = value {
    // 按应用定义的 KV 编码解码 bytes
}
```
