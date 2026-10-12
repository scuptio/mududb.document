# Update

## Syntax

```sql
UPDATE table_name
SET column_name = expression [, column_name = expression ...]
WHERE predicate [AND predicate ...];
```

## Rules

- `WHERE` is **required**. An `UPDATE` without a `WHERE` clause is rejected by the parser.
- The right-hand side of an assignment can be a literal, a placeholder (`?`), or a self-increment/decrement expression of the form `col = col +|- <integer literal or ?>` (integer and `NUMERIC` columns only; the left operand must be the assigned column itself).
- Predicates in the `WHERE` clause are combined with `AND` only and must be equality predicates covering the complete primary key (one equality per primary-key column); non-key predicates and range predicates are not supported.
- Primary-key columns cannot be updated.

## Examples

Simple update with a literal:

```sql
UPDATE stock
SET s_quantity = 100
WHERE s_w_id = 1 AND s_i_id = 10;
```

Update using an arithmetic expression (self-increment):

```sql
UPDATE stock
SET s_quantity = s_quantity + 5
WHERE s_w_id = 1 AND s_i_id = 10;
```

Update with a placeholder:

```sql
UPDATE stock
SET s_quantity = ?
WHERE s_w_id = ? AND s_i_id = ?;
```
