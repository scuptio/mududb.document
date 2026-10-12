# open

打开一个会话或逻辑连接上下文，为后续系统调用提供事务与执行上下文。

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

## 概念签名

```text
open() -> OID
```

## 参数

无。

## 返回值

- `OID` —— 会话标识符，作为后续 `get`、`put`、`query`、`command` 等系统调用的第一个参数。

## 错误

创建会话可能因宿主错误或调用资源预算耗尽而失败。本地调用需显式配置 standalone adapter；普通宿主侧 facade 可返回 `NotImplemented`。

## 事务语义

- 打开会话不会隐式启动持久事务。
- 事务边界由宿主控制；guest 接口没有独立的 commit/rollback syscall。

## 示例

```rust
let oid = mudu_open()?;
let value = mudu_get(oid, b"accounts:42")?;
mudu_close(oid)?;
```

## 注意

- 在过程代码中，会话 `OID` 通常作为第一个参数传入，因此显式的 `open`/`close` 多见于交互式测试或独立适配器代码。
- `open` 应与 `close` 配对使用，以释放会话级资源。
