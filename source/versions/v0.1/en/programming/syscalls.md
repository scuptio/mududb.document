# Syscalls

The WIT boundary declares 23 operations: sessions (2), SQL (3), KV (4), filesystem (11), and relation access (3). The workspace Rust facade does not yet wrap KV `delete`; distinguish facade availability from host/WIT support.

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
