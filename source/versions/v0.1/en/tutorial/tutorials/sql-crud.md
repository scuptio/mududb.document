# SQL CRUD Tutorial

This tutorial shows a minimal table and read/write flow.

```sql
CREATE TABLE accounts (
  id INT PRIMARY KEY,
  owner TEXT,
  balance INT
);

INSERT INTO accounts (id, owner, balance) VALUES (1, 'Alice', 5000);
INSERT INTO accounts (id, owner, balance) VALUES (2, 'Bob', 1200);

SELECT id, owner, balance FROM accounts;

UPDATE accounts SET balance = balance + 300 WHERE id = 2;
DELETE FROM accounts WHERE id = 1;
```

SQL statements in the interactive shell must end with semicolons. Meta commands include `\q`, `\help`, and `\app <name>`.
