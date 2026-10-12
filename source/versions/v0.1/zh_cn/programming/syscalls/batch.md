# batch

将多个操作作为单个批处理请求执行。

## Rust API

```rust
// sync_api
pub fn mudu_batch(oid: OID, sql: &dyn SQLStmt, params: &dyn SQLParams) -> RS<u64> {
    /* ... */
}

// async_api
pub async fn mudu_batch(oid: OID, sql: &dyn SQLStmt, params: &dyn SQLParams) -> RS<u64> {
    /* ... */
}
```

`mudu_batch` 使用与 `mudu_command` 相同的参数与返回约定。`batch` 执行原始多语句 SQL 文本，不支持 SQL 参数：参数列表必须为空，否则运行时返回 `NotImplemented`。

## 概念签名

```text
batch(ops: [string | primitive-op], options?: { atomic: bool }) -> BatchResult
```

## 参数

- `ops` —— 具体 SQL 语句或原始操作的有序列表。不支持参数化 SQL 字符串。
- `options.atomic` —— 为 true 时，运行时会尝试将批处理作为原子组执行（全有或全无）。原子批处理可能会因存储引擎的不同而被拒绝。

## 行为

- 在当前会话和事务上下文中按顺序执行 ops。
- 如果任何操作失败且 `options.atomic` 为 true，运行时会尝试回滚批处理中的先前操作；回滚保证取决于存储支持。
- 如果为 SQL 字符串提供了参数，运行时会返回 `NotImplemented`。

## 返回值

- `BatchResult` 包含每个操作的状态条目和一个摘要（成功计数、错误条目）。

## 错误与边界情况

- `NotImplemented`：当调用者为非空参数列表提供 SQL 参数占位符时。
- `ConstraintViolation`：当变更操作违反模式或唯一性约束时返回。
- `PartialFailure`：当 `atomic=false` 时，某些操作可能成功而另一些失败；请检查每个操作的状态。

## 事务语义

- 批处理参与当前事务。
- 如果未打开事务，行为遵循宿主默认值（自动提交或每个操作提交）。

## 示例

```text
-- two concrete statements in a batch
batch([
  "INSERT INTO accounts (id, owner, balance) VALUES (3, 'Eve', 3000)",
  "UPDATE accounts SET balance = balance - 500 WHERE id = 2"
], { atomic: true });
```

## 最佳实践

- 避免向 batch 传递参数占位符；在批处理前展开参数。
- 谨慎使用原子批处理；对于复杂的多步更新，优先使用显式事务。
- 优先使用较小的批处理，以减少锁争用并改善故障隔离。
