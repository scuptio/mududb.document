# close

关闭先前打开的会话，释放会话级资源。

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

## 概念签名

```text
close(session_id: OID) -> ()
```

## 参数

- `session_id` —— 之前 `open` 调用返回的 `OID`。

## 返回值

- 成功时返回 `()`。

## 错误

- `EntityNotFound`：提供的 `OID` 不对应一个已打开的会话。

## 事务语义

- 若在 `close` 时仍有未提交事务，行为取决于宿主生命周期（自动提交或回滚）。
- 在 MuduDB 过程中，内核通常会在过程调用周围管理事务边界。

## 示例

```rust
let oid = mudu_open()?;
// ... 使用 oid ...
mudu_close(oid)?;
```

## 注意

- 过程应显式关闭它打开的会话，以避免资源泄漏。
- `close` 后不要再使用已关闭的 `OID`，否则会返回 `EntityNotFound`。
