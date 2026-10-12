# SQL 语言

MuduDB 使用一种 SQL 方言，该方言由专为其存储和执行模型设计的 tree-sitter 语法解析。该方言涵盖用于模式定义的 DDL、用于数据变更的 DML，以及用于单表查询的 SELECT。本节描述每个类别、MuduDB 接受的语法，以及当前支持的限制范围。

```{toctree}
:maxdepth: 2

ddl/index
dml/index
query
data_types/index
```

## 兼容性与陷阱

以下写法能被语法接受，但 v0.1 的引擎会静默忽略对应语义（既不报错也不生效）：

- `CREATE TABLE IF NOT EXISTS` — `IF NOT EXISTS` 被忽略：表已存在时仍然报错。
- 列级 `DEFAULT` 约束 — 被解析但被忽略，不会为列填入默认值。
- 表级 `FOREIGN KEY ... REFERENCES` 约束 — 被解析但不强制执行。
- `SELECT DISTINCT` — `DISTINCT` 被忽略，返回结果不去重。
