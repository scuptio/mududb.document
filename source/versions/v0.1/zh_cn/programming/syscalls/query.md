# query

执行 SQL `SELECT` 语句，返回结果集并将行映射为 Rust 结构体（R2O，关系到对象映射）。

## Rust API

```rust
// sync_api
pub fn mudu_query<R: Entity>(
    oid: OID,
    sql: &dyn SQLStmt,
    params: &dyn SQLParams,
) -> RS<EntitySet<R>> {
    /* ... */
}

// async_api
pub async fn mudu_query<R: Entity>(
    oid: OID,
    sql: &dyn SQLStmt,
    params: &dyn SQLParams,
) -> RS<EntitySet<R>> {
    /* ... */
}
```

## 概念签名

```text
query<R: Entity>(sql: string, params?: [Value]) -> EntitySet<R>
```

## 参数

- `oid` —— 传入过程的当前系统会话标识。
- `sql` —— 单条 `SELECT` 语句，使用 `?` 作为参数占位符。
- `params` —— 绑定到占位符的值。

## 行为

- 在当前会话中执行 `SELECT`，并自动将每个结果行映射为实体类型 `R`（R2O 映射）；`R` 必须实现 `Entity` trait（见 [Mudu 过程](../mudu_procedure.md)）。
- 行可见性遵循当前事务的隔离规则，与其他读取一致。

## 返回值

- `RS<EntitySet<R>>` —— 映射后的结果集。调用 `next_record()` 逐行迭代：它返回 `RS<Option<R>>`，每行产生 `Ok(Some(row))`，结果集结束时返回 `Ok(None)`。

## 错误

- `Parse`：SQL 文本无法解析。
- `EntityNotFound`：`oid` 不对应一个存活会话。
- `Database`：存储引擎执行语句失败。

## 事务语义

- 查询参与当前会话事务。
- 一次过程调用作为一个事务运行：返回 `Ok` 提交，返回 `Err` 回滚；见 [Mudu 过程](../mudu_procedure.md)。

## 示例

```rust
use sys_interface::sync_api::mudu_query;

let wallets = mudu_query::<Wallets>(
    oid,
    sql_stmt!(&Wallets::SQL_GET_BY_PK),
    sql_params!(&(user_id,)),
)?;
if let Some(wallet) = wallets.next_record()? {
    // wallet 是由选中行映射得到的 Wallets 结构体
}
```

## 注意

- `query` 仅用于 `SELECT`。`INSERT` / `UPDATE` / `DELETE` 请使用 `command`；一次调用执行多条语句请使用 `batch`。
- 实体模块通常由 `mgen entity` 生成，同时生成上面用到的 `SQL_GET_BY_PK` / `SQL_INSERT` / `SQL_DELETE_BY_PK` SQL 文本常量。
