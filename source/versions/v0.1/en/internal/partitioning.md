# Partitioning and Worker Mapping

## Overview

Partitioning divides application data and traffic into logical partitions (partition_oid) that are mapped to backend workers (worker_oid). The goal is to exploit locality: keep related data and operations on the same worker to reduce cross-worker communication, lower latency, and improve cache locality.

## Design rationale

- Locality: co-locate related keys/records to reduce remote calls and state synchronization.
- Load isolation: partitioning limits hotspots to specific workers and simplifies load balancing by moving partitions.
- Predictable routing: clients can route requests to the partition owner, avoiding unnecessary gateway hops.

## Terminology

- partition_oid: logical identifier for a partition.
- worker_oid: backend worker process identifier that hosts partitions.
- sharding port: a TCP port on which a worker accepts connections dedicated to specific shards/partitions.
- topology: management HTTP API data that maps partition_oid -> worker_oid and provides worker addresses.

## Model & syntax

- Partition identifiers are opaque integers used in the topology mapping.
- Connection strings can embed app-level hints; topology is discovered via `http_addr` from `MUDU_CONNECTION`.

Example connection: `mudud://127.0.0.1:9527/ycsb?http_addr=127.0.0.1:8300`

Clients obtain topology and resolve partition_oid -> worker_oid, then open sessions targeting the worker OID.

## How routing works 

1. Client fetches topology from management HTTP API (http_addr).
2. Client computes or is given a partition_oid for each operation (e.g., by hashing a key or using app semantics).
3. Client looks up the partition_oid -> worker_oid mapping in the topology cache.
4. Client connects to the worker's sharding port (or standard port with worker routing) and opens a session for that worker_oid (session open includes worker_id in the config handshake).
5. Requests routed to the worker are executed locally or proxied efficiently by the worker.



## Discovering topology 

- Use the HTTP management API `GET /mudu/server/topology` or `mcli --http-addr <addr> server-topology` to retrieve live topology.
- The topology JSON contains fields:
  - `worker_count`
  - `tcp_multi_port` (bool)
  - `tcp_base_listen_port` (int)
  - per-worker `worker_index`, `worker_id`, `tcp_listen_port`, and the `partitions` list (partition ids placed on that worker).

- To resolve a partition key to a worker:
  1. Call `partition-route` with the rule name and key tuple to obtain `partition_id` and `worker_id`.
  2. Look up the worker's `tcp_listen_port` from `server-topology` result.
  3. Connect to `listen_ip:tcp_listen_port` and open a session with `worker_id` in session config.

Examples:

```bash
mcli --http-addr 127.0.0.1:8300 partition-route --rule-name r_orders --key 1001,50001
mcli --http-addr 127.0.0.1:8300 server-topology
```

## Backend mapping & caching

- Topology is authoritative on the management HTTP endpoint. Clients should cache the mapping locally and refresh periodically or on error.
- Cache invalidation: on RPC errors indicating "NoSuchPartition" or similar, clients should invalidate relevant entries and re-fetch topology.
- Update strategy: periodic short-interval refresh (seconds) plus event-driven refresh on errors.

## Sharding port and worker connection

- Workers may expose a sharding port bound to a given partition range or accept connections with a worker_id handshake.
- When supported, the client can connect directly to a worker's sharding port to reach the partitions owned by that worker.
- If multi-port per worker is enabled (tcp_multi_port), the server opens per-worker listeners to reduce accept contention and provide direct routes.

## Obtaining topology and connecting directly

- The management HTTP API (http_addr) provides a JSON topology mapping. Example flow:
  - GET /mudu/server/topology -> returns list of workers and partition assignments.
  - Extract worker network address and sharding port for target worker_oid.
  - Connect to worker address and open session with session_open_config_json containing worker_id.

## Usage examples

- Hash-based partitioning (client-side): compute partition_oid = hash(key) % partition_count, then route to topology[partition_oid].
- App-assigned partitioning: application assigns partition_oid based on business logic (player_id → partition_oid).

## Failure modes & fallbacks

- Stale mapping: on connect/refuse, refresh topology and retry with updated mapping.
- Worker failover: if a worker is down, topology will reassign partitions — clients must re-resolve and reconnect.
- Conservative fallback: if topology unavailable, client may route through management/gateway worker that proxies requests, accepting higher latency.

## Operational concerns

- Re-sharding: moving partitions between workers requires updating topology and draining connections. Clients must handle transient failures during re-sharding.
- Monitoring: expose partition-to-worker mappings and per-partition metrics to detect hot partitions.

## Performance and tuning

- Partition granularity: smaller partitions increase routing resolution but add metadata overhead.
- Connection reuse: keep persistent session per worker_oid to amortize session open cost.
- async_session_loop: use adapter async loop for high concurrency to avoid blocking on session management.

## Troubleshooting

- Common errors: NoSuchPartition, ConnectionRefused, NotAuthorized.
- Diagnostics: fetch topology, verify partition assignment, confirm worker network reachability, check sharding port availability.


---

End of partitioning document (EN).
