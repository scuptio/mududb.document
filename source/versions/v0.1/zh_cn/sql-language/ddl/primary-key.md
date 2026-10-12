# 主键

MuduDB 中的表以主键作为行标识。每个表都应声明主键：没有主键的表可以创建，但无法插入行。主键可以是：

1. **行内** — 直接在单个列上声明：
   ```sql
   id INT PRIMARY KEY
   ```

2. **表级别** — 在所有列之后声明，用于复合键：
   ```sql
   PRIMARY KEY (col_a, col_b)
   ```

主键决定行的唯一性。对于分区表，分区键（`PARTITION BY GLOBAL RULE ... REFERENCES` 引用的列）取自主键列。标记为 `PRIMARY KEY` 的列隐式为 `NOT NULL`。
