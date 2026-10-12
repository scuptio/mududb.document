# put

向当前系统会话的键值存储写入一个键值对。底层系统调用名为 `mudu_put`。

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

## 参数

- `session_id` —— `mudu_open` 返回的会话 ID。
- `key` —— 原始键字节。
- `value` —— 原始值字节。

## 行为

- 写入或覆盖会话级键值存储中的一个键值对。

## 返回值

- `RS<()>` —— 成功返回空。

## 错误

- 无效会话或宿主/后端失败通过 `MuduError` 返回；原始 KV 值不按表 schema 校验。

## 示例

```rust
mudu_put(oid, b"accounts:42", b"{\"balance\":1000}")?;
```
