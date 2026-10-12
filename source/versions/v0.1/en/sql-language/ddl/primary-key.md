# Primary Key

A MuduDB table is identified by its primary key. Every table is expected to declare one: a table without a primary key can be created, but rows cannot be inserted into it. The primary key can be:

1. **Inline** — declared directly on a single column:
   ```sql
   id INT PRIMARY KEY
   ```

2. **Table-level** — declared after all columns for composite keys:
   ```sql
   PRIMARY KEY (col_a, col_b)
   ```

The primary key determines row uniqueness. For partitioned tables, the partitioning key (the columns referenced by `PARTITION BY GLOBAL RULE ... REFERENCES`) is drawn from the primary-key columns. A column marked `PRIMARY KEY` is implicitly `NOT NULL`.
