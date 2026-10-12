# Syscalls

MuduDB procedures call into the kernel via a small set of controlled syscalls. These pages document each syscall's purpose, expected inputs, and transaction/consistency implications.

The SQL `query` syscall maps result rows onto `Entity` structs; the KV store supports point reads and writes, range scans, and deletion. The `mudu_fs_*` filesystem syscall family is documented under [Filesystem Object Types](filesystem.md). The host additionally implements `relation_get` / `relation_update` / `relation_insert` (row access that bypasses SQL parsing) and a `fetch` cursor that pages large query results; these are not yet covered by dedicated pages.

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
