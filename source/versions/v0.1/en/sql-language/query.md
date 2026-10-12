# Query

MuduDB supports single-table `SELECT` queries with a `WHERE` clause for filtering rows. The query engine evaluates predicates and returns a result set to the caller.

## 3.1 Overview

A query in MuduDB is executed in the context of the current transaction. All reads are consistent with the transaction's isolation level. Results are returned as a row set that the client can iterate over.

The current query capability is intentionally scoped to single-table lookups and scans. Joins, subqueries, and `GROUP BY`-style grouped analytics are not supported in v0.1.

## 3.2 Select

### Syntax

```sql
SELECT select_term [, select_term ...]
FROM table_name
[WHERE predicate [AND predicate ...]];
```

### Select Terms

A `select_term` is one of:

- `column_name` — select a single column.
- A single-argument aggregate call — `COUNT(*)`, `COUNT(column)`, `SUM(column)`, `AVG(column)`, `MIN(column)`, `MAX(column)`.

Each select term can carry an optional alias (`AS alias`). Without `GROUP BY`, the select list must be all plain columns or all aggregate calls; mixing the two is not supported.

### WHERE Predicates

Each predicate compares a column with a value (a literal or a `?` placeholder):

```
column_name comparison_operator value
```

The reversed form (value on the left, column on the right) is also accepted; the parser normalises it to column-on-the-left.

**Comparison operators:**

| Operator | Meaning |
|---|---|
| `=` | Equal |
| `!=` | Not equal |
| `<` | Less than |
| `<=` | Less than or equal |
| `>` | Greater than |
| `>=` | Greater than or equal |

Predicates are combined with the `AND` logical connective only. `OR` and `NOT` are not supported in v0.1.

How a predicate is executed depends on the columns involved:

- Equality predicates covering a **left prefix** of the primary key (e.g. `w = 1` or `w = 1 AND d = 2` for a primary key `(w, d, o)`) use a primary-key point or prefix lookup; equality predicates on non-prefix key columns only (e.g. `d = 2`) are rejected.
- Range comparisons (`>`, `>=`, `<`, `<=`) on primary-key columns use a primary-key range scan; however, equality and range predicates on key columns cannot be mixed (e.g. `w = 1 AND d > 2` is rejected).
- Predicates on non-key columns are evaluated row by row after the key access and can be freely combined with key predicates.
- `!=` cannot be used on primary-key columns (it is allowed on non-key columns).

### Expressions

An operand in a predicate can be:

- A column reference
- A numeric or string literal
- A placeholder (`?`; named parameters `:name` resolved by the server adapter are also available, see {doc}`dml/index`)

### Examples

Select columns from a table:

```sql
SELECT id, name FROM users;
```

Select specific columns with a filter:

```sql
SELECT id, name FROM users WHERE id = 1;
```

Select with multiple predicates:

```sql
SELECT ol_i_id, ol_amount
FROM order_line
WHERE ol_w_id = 1 AND ol_d_id = 2 AND ol_o_id = 100;
```

Select with placeholders (prepared statement):

```sql
SELECT s_i_id, s_quantity FROM stock WHERE s_w_id = ? AND s_i_id = ?;
```

Select with a reversed comparison (parser normalises the shape):

```sql
SELECT id FROM users WHERE 7 > id;
-- internally normalised to: id <= 7
```

Select with aggregate calls in the term list:

```sql
SELECT COUNT(*) AS cnt, SUM(ol_amount) FROM order_line WHERE ol_w_id = 1;
```

## Unsupported Query Features

The following are not available in v0.1:

- `JOIN` (inner, outer, cross)
- `GROUP BY` / `HAVING`
- `ORDER BY`
- `LIMIT` / `OFFSET`
- `DISTINCT`
- The `*` wildcard (`SELECT *`); list columns explicitly
- Subqueries (`SELECT ... FROM (SELECT ...)`)
- Window functions
- Common Table Expressions (`WITH`)
- Set operations (`UNION`, `INTERSECT`, `EXCEPT`)
