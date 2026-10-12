# 系统调用

WIT 边界声明 23 个操作：会话 2 个、SQL 3 个、KV 4 个、文件系统 11 个、relation 访问 3 个。仓库内 Rust facade 尚未封装 KV `delete`，请区分 facade 可用性与宿主/WIT 支持。

```{toctree}
:maxdepth: 1

syscalls/open
syscalls/close
syscalls/query
syscalls/command
syscalls/batch
syscalls/get
syscalls/put
syscalls/delete
syscalls/range
syscalls/fs
syscalls/relation
```
