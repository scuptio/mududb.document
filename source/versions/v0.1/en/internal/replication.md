# Replication Architecture

Replication is implemented by the `repl` and `mudu_raft` crates, with server integration in `server` and `mudu_runtime`. Worker WAL streams carry data; Raft agrees on commit vectors rather than re-encoding SQL operations. The committed vector bounds follower apply and leader recovery.

For deployment modes, configuration, snapshots, membership changes, observability, failover and limitations, see [Replication and Raft Fault Tolerance](../administration/replication.md). Replication currently requires the Tokio backend.
