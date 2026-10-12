# 查询

MuduDB 支持单表 `SELECT` 查询，并可通过 `WHERE` 子句过滤行。查询引擎计算谓词并将结果集返回给调用方。

## 3.1 概述

MuduDB 中的查询在当前事务上下文中执行。所有读取都与事务的隔离级别保持一致。结果以行集形式返回，客户端可以对其进行迭代。

当前的查询能力有意限定为单表查找和扫描。连接、子查询和 `GROUP BY` 分组分析在 v0.1 中不受支持。

## 3.2 Select

### 语法

```sql
SELECT select_term [, select_term ...]
FROM table_name
[WHERE predicate [AND predicate ...]];
```

### Select 项

`select_term` 可以是以下之一：

- `column_name` — 选择单个列。
- 单参数聚合函数调用 — `COUNT(*)`、`COUNT(column)`、`SUM(column)`、`AVG(column)`、`MIN(column)`、`MAX(column)`。

每个 select 项可以带可选别名（`AS alias`）。不使用 `GROUP BY` 时，选择列表必须全部是普通列或全部是聚合调用，二者不能混合。

### WHERE 谓词

每个谓词将一个列与一个值（字面量或占位符 `?`）进行比较：

```
column_name comparison_operator value
```

也接受值在左、列在右的形式；解析器会将其规范化为列在左的形式。

**比较运算符：**

| 运算符 | 含义 |
|---|---|
| `=` | 等于 |
| `!=` | 不等于 |
| `<` | 小于 |
| `<=` | 小于或等于 |
| `>` | 大于 |
| `>=` | 大于或等于 |

谓词仅通过 `AND` 逻辑连接符组合。`OR` 和 `NOT` 在 v0.1 中不受支持。

谓词的执行方式取决于涉及的列：

- 覆盖主键**左前缀**的等值谓词（例如主键为 `(w, d, o)` 时的 `w = 1` 或 `w = 1 AND d = 2`）走主键点查或前缀查找；仅给出非前缀主键列的等值谓词（例如 `d = 2`）会被拒绝。
- 主键列上的范围比较（`>`、`>=`、`<`、`<=`）走主键范围扫描；但主键等值谓词与主键范围谓词不能混合（例如 `w = 1 AND d > 2` 会被拒绝）。
- 非主键列上的谓词在键访问之后逐行过滤，可与主键谓词自由组合。
- `!=` 不能用于主键列（非主键列可以）。

### 表达式

谓词中的操作数可以是：

- 列引用
- 数值或字符串字面量
- 占位符（`?`；也可以使用由服务器适配层解析的命名参数 `:name`，见 {doc}`dml/index`）

### 示例

从表中选择列：

```sql
SELECT id, name FROM users;
```

选择特定列并带过滤条件：

```sql
SELECT id, name FROM users WHERE id = 1;
```

选择多个谓词：

```sql
SELECT ol_i_id, ol_amount
FROM order_line
WHERE ol_w_id = 1 AND ol_d_id = 2 AND ol_o_id = 100;
```

选择带占位符（预备语句）：

```sql
SELECT s_i_id, s_quantity FROM stock WHERE s_w_id = ? AND s_i_id = ?;
```

选择带反向比较（解析器会规范化其形式）：

```sql
SELECT id FROM users WHERE 7 > id;
-- internally normalised to: id < 7
```

选择列表中使用聚合函数调用：

```sql
SELECT COUNT(*) AS cnt, SUM(ol_amount) FROM order_line WHERE ol_w_id = 1;
```

## 不支持的查询功能

以下功能在 v0.1 中不可用：

- `JOIN`（内连接、外连接、交叉连接）
- `GROUP BY` / `HAVING`
- `ORDER BY`
- `LIMIT` / `OFFSET`
- `DISTINCT`
- `SELECT *` 通配符（需显式列出列名）
- 子查询（`SELECT ... FROM (SELECT ...)`）
- 窗口函数
- 公用表表达式（`WITH`）
- 集合运算（`UNION`、`INTERSECT`、`EXCEPT`）
