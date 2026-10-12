# 数据定义

MuduDB 支持一个精简的 DDL 子集，用于创建和删除表。模式变更通过标准的 `CREATE TABLE` 和 `DROP TABLE` 语句执行。分区规则和放置位置也可以为分布式表声明。

## 表基础

MuduDB 中的表是一个命名的行集合。每一行具有在创建时定义的相同列集合。MuduDB 在当前版本中不支持 `ALTER TABLE`；模式在 `CREATE TABLE` 成功后即固定不变。

表在当前会话的默认数据库上下文中创建。不存在 `CREATE DATABASE` 或 `USE database` 语法；会话本身决定逻辑数据库范围。

除 `CREATE TABLE` 只接受裸表名外，各语句中的表名都可以写成模式限定形式 `schema.table`（例如 `shop.users`）。不带模式前缀的引用先按会话的当前模式解析，未命中时回退到默认模式 `mududb`；`schema.table` 形式的引用精确匹配、不回退。`CREATE TABLE` 总是把新表创建在会话的当前模式中。

```{toctree}
:maxdepth: 1

columns
create-table
primary-key
drop-table
```

## 文件系统类型

`CREATE TYPE FILESYSTEM` 注册一个命名的文件系统类型，之后可作为列类型在 `CREATE TABLE` 中使用；该列物理上存储一个 128 位对象标识（`U128`），指向文件或目录对象。

```sql
CREATE TYPE FILESYSTEM FILE photo_fs;
CREATE TYPE FILESYSTEM DIRECTORY dir_fs;
DROP TYPE photo_fs;
```

类型名只能包含字母、数字和下划线，且不能以数字开头。文件与目录对象的读写接口见 {doc}`../../programming/filesystem`。

## 不支持的 DDL

以下 DDL 功能在 v0.1 中不可用：

- `ALTER TABLE`（添加 / 删除 / 重命名列）
- `CREATE INDEX`
- `CREATE VIEW`
- 外键约束（`FOREIGN KEY ... REFERENCES`）
- `ON DELETE CASCADE` / `ON UPDATE`
- `CHECK` 约束
- `UNIQUE` 约束（仅 `PRIMARY KEY` 被强制执行）
