# Procedure Runtime

The procedure runtime hosts Mudu Procedures inside the database process. It loads WebAssembly component modules, binds them to the kernel's syscall surface, and executes procedure calls under kernel control.

## Design

The runtime is passive: it does not define its own scheduler or execution policy. The kernel decides when and where a procedure runs, and the runtime provides the execution container.

A typical invocation flow:

1. The kernel receives a procedure invocation request.
2. The kernel routes the request to the worker that owns the target partition.
3. The worker's runtime loads the app/module if it is not already resident.
4. The runtime calls the exported procedure function with the session `OID` and user arguments.
5. Inside the procedure, database operations trap into the kernel through the syscall interface.
6. The kernel applies transaction, scheduling, and consistency rules to each syscall.
7. When the procedure returns, the kernel commits or rolls back the transaction and returns the result.

## Synchronous source, asynchronous execution

Developers write procedures using synchronous-looking APIs (`sys_interface::sync_api`). At build time, the `mtp` transpiler can rewrite blocking syscalls into async-compatible forms suitable for the runtime's continuation-driven executor.

This means:

- Procedure authors keep a sequential programming model.
- The runtime still yields while waiting for I/O or remote RPCs.
- The same source can run interactively during development and be deployed as a stored procedure.

## Syscall boundary

Procedures access database state through a compact syscall set:

- Session: `mudu_open`, `mudu_close`
- SQL: `mudu_query`, `mudu_command`, `mudu_batch`
- KV: `mudu_get`, `mudu_put`, `mudu_range`

Each syscall carries the session `OID` so the kernel can enforce transactional and scheduling context.

## Use cases

This model suits applications that need:

- strong consistency inside a single procedure boundary,
- low-latency business logic close to the data,
- data-local execution that avoids repeated client-server round trips.

See [Mudu Procedure](../programming/mudu_procedure.md) for the programming model, [System Call Reference](../programming/syscalls.md) for the syscall API, and [WebAssembly Integration](wasm.md) for the component runtime details.


See [Suspendable Procedure Execution](suspendable-execution.md) for fresh Store creation, bounded synchronous execution and cancellation/commit boundaries. The interface also includes [filesystem](../programming/syscalls/fs.md) and [relation](../programming/syscalls/relation.md) operations; low-level WIT signatures differ from the workspace Rust facade.
