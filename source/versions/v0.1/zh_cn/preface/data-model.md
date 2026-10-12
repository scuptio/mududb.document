# 数据模型

MuduDB 同时暴露 SQL 风格的关系型数据访问与面向过程的业务逻辑。本页描述两条路径共享的数据模型。

## 表与主键

MuduDB 数据库包含通过 SQL DDL 定义的**表**。每张表具有：

- 表名与逻辑 `table_id`。
- 一个或多个具有声明数据类型与可空性的列。
- 一个**主键**，可以是单列或复合主键。

主键决定物理聚簇顺序，并用于点查、范围扫描与分区路由。

## 索引

当前主键是主要访问路径。二级索引支持正在规划中；在此之前，主键范围扫描与直接键访问是主要查询机制。

## 分区键

表可以绑定到全局**分区规则**。分区列必须构成主键的前缀。v0.1 使用 RANGE 分区：每个分区拥有一个连续的键区间。

分区规则、绑定、placement 与路由模型详见[分区](../internal/partitioning.md)。

## 过程输入与输出类型

Mudu 过程是操作 MuduDB 数据的普通函数。其输入与返回类型必须能被 MuduDB 的 datum / tuple 转换系统表示：

- 标量值：`i32`、`i64`、`String`、`f32`、`f64` 等。
- 容器值：`Vec<T>`、元组以及生成的 `Entity` 结构体。

具体支持的类型面由当前 `mudu_type` 转换与转译器实现定义。

## Schema 到代码的映射

`mgen` 工具读取 `ddl.sql` 并生成实现 `Entity` trait 的 Rust 结构体，从而提供类型化的查询结果与参数：

```rust
#[derive(Debug, Clone, Default)]
struct Account {
    id: i64,
    owner: String,
    balance: i64,
}

impl Entity for Account { /* 由 mgen 生成 */ }
```

`mudu_query::<Account>(oid, sql, params)` 会自动将结果行映射为 `Account` 实例。

## 兼容性

当数据模型语义发生变化时（例如新增格式版本或修改 tuple 编码），MuduDB 通过版本化契约跟踪兼容性。应用代码与持久化数据应按相关契约的升级/回滚规则进行迁移。

详见[格式与协议契约](../reference/contracts/index.md)。
