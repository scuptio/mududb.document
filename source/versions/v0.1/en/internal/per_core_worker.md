# Per-core Worker Model

Kernel modes (`Tokio` and Linux `IOUring`) run worker threads with local execution state and asynchronous I/O. `worker_threads = 0` selects available parallelism; an explicit positive value selects the worker count. This worker count does not by itself promise CPU pinning.

Each worker owns session/transaction state and services data operations for its assigned partitions. Async tasks yield while awaiting I/O or internal RPCs. Local operations stay on the owning worker; partition routing can forward remote operations through the internal message bus.

With more than one worker, the server enables consecutive per-worker TCP ports starting at `tcp_listen_port`. Clients can discover worker identifiers, ports and partition placement through `mcli server-topology`, then open a session targeting the selected worker. `routing_mode` controls connection routing; it does not define a SQL hash-partitioning scheme.

See [Partitioning](partitioning.md) for explicit range rules and placements, [Suspendable Procedure Execution](suspendable-execution.md) for guest scheduling, and [Replication](replication.md) for node-level failover. Worker locality does not imply automatic live partition migration or distributed atomic commit.

Implementation: `crates/db-kernel/server/src/server/serve/`, `crates/db-kernel/worker/src/`, and the `mudu_runtime` backend startup code.
