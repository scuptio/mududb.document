# 首次会话

## 启动服务器

`mudud` 需要配置文件，且不会自动创建；如果尚不存在，请先生成默认的 `./mudud.cfg`：

```bash
mudud init-cfg
ulimit -n 65535
mudud
```

在 `systemd` 下运行时，请在服务级别配置 `LimitNOFILE=65535`。

## 打开交互式 SQL Shell

```bash
mcli --addr 127.0.0.1:9527 shell --app demo
```

运行完整的 CRUD 流程：

```sql
CREATE TABLE users_demo (
  id INT PRIMARY KEY,
  name TEXT
);

INSERT INTO users_demo (id, name) VALUES (1, 'Alice');
SELECT id, name FROM users_demo WHERE id = 1;

UPDATE users_demo SET name = 'Alice-Updated' WHERE id = 1;
DELETE FROM users_demo WHERE id = 1;
```

退出 shell：

```text
\q
```

## 下一步

继续阅读 {doc}`../tutorials/wallet`，以构建、安装和调用过程应用程序。
