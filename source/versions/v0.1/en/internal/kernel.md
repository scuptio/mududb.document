# Kernel

The MuduDB kernel is the correctness foundation of the system. It owns storage, transaction processing, query execution, scheduling, and the system-call boundary that procedures use to access database state.

## Kernel-runtime split

MuduDB uses a two-layer architecture:

- **Kernel** — provides the database engine: pages, relations, indexes, WAL, transactions, query planning/execution, and scheduling.
- **Runtime** — hosts user-defined procedures as WebAssembly components and forwards their syscalls into the kernel.

The runtime is intentionally passive. It does not introduce its own scheduler or execution policy. Procedure execution is invoked and controlled by the kernel so that scheduling, correctness, and data access remain under a single authority.

## Responsibilities

| Area | What the kernel does |
|------|----------------------|
| Storage | Manages relation and time-series files, page allocation, slot arrays, and on-disk formats. |
| Transactions | Provides session contexts, conflict detection, commit/rollback, and WAL persistence. |
| Query execution | Parses and executes SQL queries, performs partition routing, and returns result sets. |
| Scheduling | Assigns work to per-core workers and coordinates continuation-driven async execution. |
| Syscalls | Exposes a narrow system-call surface (`mudu_open`, `mudu_query`, `mudu_get`, ...). |
| Compatibility | Tracks format versions and routes persistent data through upgrade/rollback handlers. |

## Worker model

The kernel organizes execution around per-core workers. Each worker owns a subset of partitions, local connection state, and (in `IOUring`/`Tokio` mode) a local async runtime. This keeps hot-path data on the same core and reduces cross-core contention.

See [Per-core worker model](per_core_worker.md) and [Partitioning](partitioning.md) for more details.

## Boundaries and compatibility

The kernel's on-disk and wire formats are governed by versioned contracts in the [Reference](../reference/contracts/index.md) section. User-facing documentation should describe subsystem boundaries without turning implementation details into compatibility promises.

## Source layout

The repository root has no `Cargo.toml`. `crates/common`, `crates/db-kernel`, `crates/sdk`, and `crates/tools` are independent workspaces; `example/mpk`, `crates/sdk/mudu_api/rust`, and `crates/bindings` have separate build contexts. The native adapter ABI workspace uses an isolated target directory.

`mudu_kernel` is a facade over fourteen crates: `catalog`, `index`, `wal`, `xengine`, `storage`, `meta`, `command`, `sql`, `conn`, `xcontract`, `worker`, `repl`, `backup`, and `server`. `runtime_wasm` hosts components, while `mudu_runtime` integrates serving and management. `mudu_raft` implements consensus. See `ARCHITECTURE.md` in the source repository for dependency rules.

Run a focused check from the relevant workspace, for example `cargo test --manifest-path crates/db-kernel/Cargo.toml --workspace`. Root-level orchestration uses `cargo make`, `cargo make check`, and `cargo make test`.
