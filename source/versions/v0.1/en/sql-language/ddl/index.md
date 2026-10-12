# Data Definition

MuduDB supports a concise DDL subset for creating and dropping tables. Schema changes are executed through the standard `CREATE TABLE` and `DROP TABLE` statements. Partition rules and placements can also be declared for distributed tables.

## Table Basics

A table in MuduDB is a named collection of rows. Each row has the same set of columns defined at creation time. MuduDB does not support `ALTER TABLE` in the current release; the schema is fixed after `CREATE TABLE` succeeds.

Tables are created within the default database context of the current session. There is no `CREATE DATABASE` or `USE database` syntax; the session itself determines the logical database scope.

Except for `CREATE TABLE`, which only accepts a bare table name, table names in any statement may be written in schema-qualified form `schema.table` (e.g. `shop.users`). An unqualified reference resolves against the session's current schema first and falls back to the default schema `mududb`; a `schema.table` reference matches exactly and never falls back. `CREATE TABLE` always creates the new table in the session's current schema.

```{toctree}
:maxdepth: 1

columns
create-table
primary-key
drop-table
```

## Filesystem Types

`CREATE TYPE FILESYSTEM` registers a named filesystem type that can then be used as a column type in `CREATE TABLE`; the column physically stores a 128-bit object identifier (`U128`) referring to a file or directory object.

```sql
CREATE TYPE FILESYSTEM FILE photo_fs;
CREATE TYPE FILESYSTEM DIRECTORY dir_fs;
DROP TYPE photo_fs;
```

Type names may contain only letters, digits, and underscores, and must not start with a digit. See {doc}`../../programming/filesystem` for the file and directory access interfaces.

## Unsupported DDL

The following DDL features are not available in v0.1:

- `ALTER TABLE` (add / drop / rename columns)
- `CREATE INDEX`
- `CREATE VIEW`
- Foreign-key constraints (`FOREIGN KEY ... REFERENCES`)
- `ON DELETE CASCADE` / `ON UPDATE`
- `CHECK` constraints
- `UNIQUE` constraints (only `PRIMARY KEY` is enforced)
