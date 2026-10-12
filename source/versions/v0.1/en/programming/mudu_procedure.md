# Mudu Procedure

## 1. Design goals

Mudu Procedure unifies interactive execution and stored procedures into a single, data-local programming model. Goals:

- Data locality: execute business logic where data lives to minimize cross-worker communication.
- Developer ergonomics: allow implementing procedures in modern languages and familiar sequential code patterns.
- Safety and isolation: run user logic in a WASI sandbox and restrict access through explicit syscalls.
- Performance: enable low-latency, high-concurrency workloads via async runtimes and per-worker affinity.

## 2. Interactive execution vs stored procedures — and the unified model

Traditional interactive execution (REPL / client-driven) is immediate and flexible but often incurs remote calls and higher latency. Stored procedures are pre-deployed server-side code that run close to data but are typically constrained to specialized languages and deployment models.

Mudu Procedure unifies both:

- Same procedure code can be invoked interactively from a client (ad-hoc call) or installed and invoked like a stored procedure.
- The runtime supports fine-grained syscalls that let procedure code perform database operations without leaving the sandbox.
- This means developers can prototype interactively, then publish the same code as a deployed procedure that runs with data locality and operational controls.

The following diagrams show the two execution models and how Mudu unifies them into a single WASI-based execution flow.

**Interactive invocation** (left): the client sends individual queries and commands, with network round-trips for each step.

**Stored procedure deployment** (right): the application sends a single request; all query execution, logic, and commit happen data-local inside the database runtime.

::::{raw} html
<div style="display: flex; gap: 1rem; flex-wrap: wrap;">
  <div style="flex: 1; min-width: 280px;">
    <img src="../_static/interactive_tx.png" alt="Interactive execution flow — client-driven round trips" style="max-width: 100%; border: 1px solid #ddd;">
  </div>
  <div style="flex: 1; min-width: 280px;">
    <img src="../_static/procedural_tx.png" alt="Stored procedure deployment — data-local execution" style="max-width: 100%; border: 1px solid #ddd;">
  </div>
</div>
::::

Both converge into a single WASI-based execution flow where procedures call syscalls and the kernel coordinates transaction and routing.

## 3. WebAssembly / WASI compilation and runtime

Mudu compiles procedures to WebAssembly (WASI) components and executes them inside a sandboxed runtime.

Why WASI?

- Portability: WASI modules are platform-neutral and language-agnostic.
- Sandboxing: the kernel controls syscalls and I/O surface to enforce safety and resource limits.
- Tooling: many languages can target WASI, enabling multi-language support.

Typical toolchain flow:

1. Author procedure in a supported language (this guide uses the Rust facade).
2. Run `mtp` to generate wrappers and apply the selected sync/async transformation.
3. Compile the generated source to a WASM component; use the example build tasks to preserve this order.
4. Deploy the module (put the mpk/package into mpk_path) and register it with the kernel.
5. Invoke interactively or as stored procedure; the kernel handles syscalls, transactions, and routing.

The runtime exposes a compact syscall set (open session, get/put/range/query/command/batch, etc.). Procedures use these syscalls to interact with database state; all network and storage access is mediated by the kernel.

## 4. Procedure specification

A Mudu Procedure is a function marked with the `/**mudu-proc**/` marker, which tells the transpiler to expose the following function as a procedure:

```text
/**mudu-proc**/
fn {procedure_name}(
    oid: OID,
    {argument_list...}
) -> RS<{return_value_type}>
```

- **Procedure name** — a valid Rust function name.
- **`oid`** — the current system session ID passed into the procedure.
- **Arguments** — each argument must be representable by Mudu's datum / tuple conversion system. Commonly used forms include scalars such as `i32`, `i64`, `String`, `f32`, and `f64`, as well as container forms such as `Vec<String>`; the exact supported surface is defined by the `mudu_type` conversion and the transpiler rather than a fixed whitelist.
- **Return value** — must be representable by the same conversion system; current implementations support scalar values, tuple return values, and `Vec<T>` style result containers.

The result alias `RS` used by procedure code is defined as:

```rust
use mudu::error::err::MError;
pub type RS<X> = Result<X, MError>;
```

Inside a procedure, code interacts with the database through the [syscalls](../programming/syscalls.md): SQL syscalls (`mudu_query` / `mudu_command` / `mudu_batch`) and key/value syscalls (`mudu_get` / `mudu_put` / `mudu_range`).

## 5. Transactions

Mudu Procedures support two transaction execution modes.

**Automatic mode (current).** Each procedure invocation runs as an independent transaction:

- the transaction commits automatically when the procedure returns `Ok`;
- it rolls back automatically when the procedure returns `Err`.

**Manual mode (planned).** Passing a shared session `oid` across multiple procedures for explicit transaction control is a planned capability. The current procedure API exposes no `commit` / `rollback`; the pseudocode below is illustrative only:

```text
procedure1(oid);
procedure2(oid);
commit(oid); // explicit commit; or rollback(oid) for explicit rollback
```

## 6. Entity mapping

`mudu_query` automatically performs R2O (relation-to-object) mapping: it returns an `EntitySet<R>` whose rows are decoded into the entity type `R`. `R` must implement the `Entity` trait:

```rust
pub trait Entity: Datum {
    fn tuple_desc() -> &'static TupleFieldDesc;

    fn table_name() -> &'static str;

    fn from_tuple(row: &TupleField) -> RS<Self>;

    fn from_tuple_value(row: &TupleValue) -> RS<Self>;

    fn to_tuple(&self) -> RS<TupleField>;

    fn to_tuple_value(&self) -> RS<TupleValue>;
}
```

Entity implementations are normally generated, not hand-written: for each table in `sql/ddl.sql`, `mgen entity` generates a Rust module containing:

- the row struct (e.g. `Wallets`) implementing `Entity`, plus `TABLE_NAME` and `columns::*` name constants;
- pre-built SQL text constants `SQL_GET_BY_PK`, `SQL_INSERT`, and `SQL_DELETE_BY_PK` for tables with a primary key, and `insert_params()` returning the typed bind tuple for `SQL_INSERT`;
- a partial-update changeset `<Table>Change` holding one `FieldChange<T>` per non-key column: `Unchanged` (the default) leaves the column out of the update, `Set(v)` sets it, and for a nullable column `Set(None)` writes a SQL `NULL`.

`update_by_pk(...)` builds the `UPDATE ... SET ... WHERE <pk> = ?` statement and its bind parameters, and returns `None` when no column is marked:

```rust
let change = WalletsChange {
    balance: FieldChange::Set(Some(200)),
    ..WalletsChange::default()
};
if let Some((sql, params)) = change.update_by_pk(user_id) {
    mudu_command(oid, sql_stmt!(&sql), sql_params!(&params))?;
}
```

## 7. Multi-language support

Unlike many traditional stored-procedure environments that bind to an embedded language (SQL/PLSQL/PLpgSQL), Mudu Procedure supports multiple high-level languages by compiling them to WASM components.

- Currently supported guest languages: Rust, AssemblyScript, C#, Python, C, and Go. This guide uses the Rust facade, the reference implementation of the procedure API.
- Future: other languages that can emit WASM components (Zig and more).

This opens the door to leveraging modern language ecosystems, package managers, and developer tools while keeping execution safe and portable.

## 8. Color-bind async/await: language ergonomics and runtime

Mudu adopts a "color-bind" model where asynchrony is a runtime feature rather than a mandatory language construct:

- Developers write sequential, synchronous-looking code for clarity and maintainability.
- The mudu toolchain (mudu-transpiler / compiler passes) rewrites or transforms blocking syscalls into async-compatible invocations (await points) as part of the build pipeline.
- This is similar in spirit to Zig's approach where the runtime can provide an async model while the source code remains simple.

Benefits:

- Developer experience: write straightforward code without manual async state machines.
- Runtime efficiency: the generated WASI module cooperates with the host async runtime (io_uring / Tokio) to achieve high concurrency.


The transpiler rewrites syscall calls into async syscall invocations under the hood so that the runtime can yield when waiting for I/O or remote RPCs.

## 9. Runtime constraints

Invocation limits are enabled by default and configured in `[invocation_limits]`: memory, fuel, wall-clock timeout, input/output sizes, host calls, resources and admission concurrency. See [Configuration Reference](../administration/configuration.md).

Automatic transaction handling describes the hosted invocation path. Native adapter calls follow the selected backend's transaction semantics; cross-worker distributed atomic commit is not implemented. Do not infer rolling upgrades, session draining, per-procedure capability policies or a complete audit trail from MPK packaging alone.

See [SDKs and Execution Modes](sdk.md) for choosing hosted versus local execution, and [Suspendable Procedure Execution](../internal/suspendable-execution.md) for async-world suspension and per-invocation isolation.

## References & examples

- See the [system call reference](../programming/syscalls.md) for `mudu_open`, `mudu_get`, `mudu_put`, `mudu_query`, `mudu_command`, and more.
- See [Filesystem Object Types](../programming/filesystem.md) for fs object DDL and the `mudu_fs_*` syscalls.
- See [WebAssembly Integration](../internal/wasm.md) for the component model and WASI sandbox.
- See [Application Packages](../administration/packages.md) for how to build and install an `.mpk` package.
- See architecture diagrams in the docs (`/_static/procedural_tx.png`) for visual flow between client, kernel, and worker.


End of Mudu Procedure overview (EN).
