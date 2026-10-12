# Data Manipulation

MuduDB supports `INSERT`, `UPDATE`, and `DELETE` for modifying table data. `COPY FROM` and `COPY TO` are also available for bulk load and export.

## Overview

All DML statements execute inside the current transaction context. If no explicit transaction is active, each statement is auto-committed. MuduDB does not support `MERGE`, `UPSERT`, or `TRUNCATE TABLE` in v0.1.

Prepared-statement placeholders (`?`) can be used in place of literal values. The kernel binds parameters at execution time.

Besides `?` positional placeholders, a statement may use named parameters `:name` (a colon followed by `[A-Za-z_][A-Za-z0-9_]*`):

- Named parameters are resolved by the server adapter before execution: the SQL is rewritten into positional form in occurrence order and values are matched by name — a repeated `:name` reuses its value at every occurrence.
- Content inside string literals (`'...'`, `"..."`), `--` line comments, `/* */` block comments, and `::type` casts is never treated as a placeholder.
- `?` and `:name` must not be mixed in one statement.
- Every named parameter in the statement must have a corresponding value, and every supplied parameter name must be used by the statement; otherwise the statement fails before execution.

```{toctree}
:maxdepth: 1

insert
update
delete
```

## COPY

Bulk load from or export to a CSV file.

```sql
COPY table_name FROM 'file_path';
COPY table_name TO 'file_path';
```

Examples:

```sql
COPY users FROM 'users.csv';
COPY users TO 'users_export.csv';
```

`COPY` does not accept a column list in v0.1; the file must match the table column order.

## Unsupported DML

The following features are not available in v0.1:

- `MERGE` / `UPSERT`
- `TRUNCATE TABLE`
- `RETURNING` clause on `INSERT` / `UPDATE` / `DELETE`
- Multi-table `UPDATE` or `DELETE`
- `ON CONFLICT` handling
