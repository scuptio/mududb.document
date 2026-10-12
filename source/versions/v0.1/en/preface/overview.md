# Overview

MuduDB is built around data-local business procedure execution.

The database kernel owns storage, transactions, query execution, and scheduling control. User procedures are hosted as WebAssembly Component Model modules and access database capabilities through controlled syscalls.

This structure reduces cross-process and cross-network traffic on critical paths while keeping correctness boundaries concentrated in the database kernel.

## Terminology

| Term | Meaning |
|------|---------|
| **Mudu Procedure (MP)** | A user-defined function that runs inside the MuduDB engine, close to the data. |
| **OID** | Object Identifier; the session/transaction handle passed as the first argument to every procedure. |
| **MPK** | MuduDB package; a ZIP archive containing DDL, descriptors, and compiled WebAssembly components. |
| **App / Module / Procedure** | Installed package hierarchy: an app contains modules, modules contain procedures. |
| **Kernel** | The database correctness foundation: storage, transactions, query execution, scheduling. |
| **Runtime** | The WebAssembly component host that runs user procedures under kernel control. |
| **Partition** | A logical shard of data mapped to a worker by a partition rule and placement metadata. |
| **Worker** | A per-core execution thread that owns a subset of partitions and local state. |
| **Primary / Replica** | In streaming replication, the primary accepts writes and pushes WAL segments; replicas are read-only and replay them. |
| **Raft** | Consensus-based fault tolerance: a group of nodes elects a leader, commits through majority, and self-heals on failure. |
| **Epoch** | A monotonically increasing generation number used to fence stale primaries during manual failover. |

## Where to go next

- [Data model](data-model.md)
- [Getting started](../tutorial/getting-started/index.md)
- [Mudu Procedure](../programming/mudu_procedure.md)
- [Kernel](../internal/kernel.md)
- [Reference](../reference/index.md)
