# The SQL Language

MuduDB uses a SQL dialect parsed by a tree-sitter grammar tailored to its storage and execution model. The dialect covers DDL for schema definition, DML for data mutation, and SELECT for single-table queries. This section describes each category, the syntax MuduDB accepts, and the limits of what is currently supported.

```{toctree}
:maxdepth: 2

ddl/index
dml/index
query
data_types/index
```

## Compatibility and Pitfalls

The following syntax is accepted by the grammar, but the v0.1 engine silently ignores its semantics (no error is raised and the meaning does not take effect):

- `CREATE TABLE IF NOT EXISTS` — `IF NOT EXISTS` is ignored: creating an existing table still fails.
- Column-level `DEFAULT` constraints — parsed but ignored; no default value is filled in for the column.
- Table-level `FOREIGN KEY ... REFERENCES` constraints — parsed but not enforced.
- `SELECT DISTINCT` — `DISTINCT` is ignored; the result is not deduplicated.
