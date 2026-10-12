# Suspendable Procedure Execution

MuduDB hosts components with synchronous or asynchronous host-call imports. Async-world calls use Wasmtime `call_async`: a guest waiting for an asynchronous host call can suspend and return `Poll::Pending`, allowing the executor to progress other invocations. A Store has at most one active invocation.

## Fresh state for each invocation

The historical type name `ProcedureInstancePool` now denotes an instance factory. Every lease creates a fresh `Store`, WASI context and component instance; only compiled component code and pre-instantiation metadata are reused. Guest linear memory, globals and handles are not retained between invocations. The caller's worker context and invocation budget are attached before instantiation.

The Store enforces memory/table limits, fuel and an epoch deadline. Async stores also yield cooperatively at a fuel interval. Admission is bounded by the global/per-application concurrency and memory budgets in [Configuration Reference](../administration/configuration.md).

## Synchronous calls

Synchronous guests run on a bounded executor with four plain worker threads and a bounded job queue. Server code awaits completion through `call_on_executor`; it does not create an unbounded thread per request or block the database executor on a thread join. An async-importing component called through this compatibility path is driven by a current-thread runtime on the executor thread.

## Transaction and cancellation lifecycle

The runtime establishes the task/session context, starts a transaction when needed, executes the component, cleans up host resources, then commits or rolls back. Failures and budget exhaustion before commit lead to rollback of the invocation-owned transaction. Once the commit boundary is crossed, cancellation no longer changes the outcome. A commit failure can have an unknown outcome and is reported as such; it must not be interpreted as a confirmed rollback.

Concurrent invocations need distinct task contexts. Native adapters and cross-worker writes have separate transaction limitations; see [Mudu Procedure](../programming/mudu_procedure.md).

## Implementation

- `crates/db-kernel/runtime_wasm/src/service/procedure_instance_pool.rs`
- `crates/db-kernel/runtime_wasm/src/service/procedure_invoke_component.rs`
- `crates/db-kernel/runtime_wasm/src/service/invocation_executor.rs`
- `crates/db-kernel/runtime_wasm/src/service/app_inst_impl.rs`
