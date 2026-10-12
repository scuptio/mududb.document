# 删除

## 语法

```sql
DELETE FROM table_name
WHERE predicate [AND predicate ...];
```

## 规则

- `WHERE` 是**必需的**。不带 `WHERE` 子句的 `DELETE` 将被解析器拒绝。
- 谓词仅通过 `AND` 组合，且必须是对完整主键的等值谓词（每个主键列一个等值条件）；不支持非主键列谓词和范围谓词。

## 示例

删除特定行：

```sql
DELETE FROM new_order
WHERE no_w_id = 1 AND no_d_id = 2 AND no_o_id = 100;
```

使用占位符删除：

```sql
DELETE FROM history
WHERE h_id = ?;
```
