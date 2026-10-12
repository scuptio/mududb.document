# 系统调用

MuduDB 过程通过一组精简的受控系统调用与内核交互。这些页面记录了每个系统调用的用途、预期输入以及事务/一致性影响。

SQL `query` 系统调用将结果行映射为 `Entity` 结构体；KV 存储支持点读、点写、范围扫描和删除。`mudu_fs_*` 文件系统调用家族见 [文件系统对象类型](filesystem.md)。宿主还实现了 `relation_get` / `relation_update` / `relation_insert`（绕过 SQL 解析的行访问）以及用于分页大结果集的 `fetch` 游标；这些尚无专门页面。

```{toctree}
:maxdepth: 1

syscalls/open
syscalls/close
syscalls/read
syscalls/write
syscalls/range-scan
syscalls/delete
syscalls/query
syscalls/batch
syscalls/command
```
