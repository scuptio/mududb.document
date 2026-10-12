# 更新

## 语法

```sql
UPDATE table_name
SET column_name = expression [, column_name = expression ...]
WHERE predicate [AND predicate ...];
```

## 规则

- `WHERE` 是**必需的**。不带 `WHERE` 子句的 `UPDATE` 将被解析器拒绝。
- 赋值的右侧可以是字面量、占位符（`?`），或形式为 `col = col +|- <整数字面量或 ?>` 的自增/自减表达式（仅整数与 `NUMERIC` 列支持；左侧必须是被赋值列本身）。
- `WHERE` 子句中的谓词仅通过 `AND` 组合，且必须是对完整主键的等值谓词（每个主键列一个等值条件）；不支持非主键列谓词和范围谓词。
- 主键列不能被更新。

## 示例

使用字面量的简单更新：

```sql
UPDATE stock
SET s_quantity = 100
WHERE s_w_id = 1 AND s_i_id = 10;
```

使用算术表达式更新（自增）：

```sql
UPDATE stock
SET s_quantity = s_quantity + 5
WHERE s_w_id = 1 AND s_i_id = 10;
```

使用占位符更新：

```sql
UPDATE stock
SET s_quantity = ?
WHERE s_w_id = ? AND s_i_id = ?;
```
