# SQL CRUD 教程

本教程展示了一个最小化的表以及读写流程。

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

交互式 shell 中的 SQL 语句必须以分号结尾。元命令包括 `\q`、`\help` 和 `\app <name>`。
