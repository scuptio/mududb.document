# Churcuring and CQL

```{note}
This chapter describes an external language project and design material. It is not an implemented MuduDB installation or API contract. The current MuduDB checkout does not ship `cqlc`; use the supported MPK/SDK paths in this guide for runnable examples. The external compiler status has not been revalidated by this documentation update.
```


## 1. Overview

Churcuring is a declarative programming toolchain targeting MuduDB. Its core language is
**CQL (Churcuring Query Language)** — a declarative, strongly typed query and business-logic
language. CQL borrows set comprehensions, quantifiers, and definition structure from TLA+,
uses Rust-style naming, and aims to compile formal specifications directly into executable,
verifiable database logic.

Key design points:

- **No JOIN, ever**: every cross-table association is an explicit `lookup` over keyed tables
  (`table = { Key → Object }`). Joins are eliminated at the language and compiler level,
  not merely discouraged.
- **Effect tiers**: all computation is pure; the only side effects are table reads
  (`read`/`lookup`) and writes (`insert`/`update`/`delete`), organized into three operator
  tiers (L0 `function` / L1 `query` / L2 `action`) that may only compose upward.
- **Total determinism**: same snapshot + same parameters ⇒ bit-identical results, including
  identical trap behavior.
- **Provable termination by default**: comprehensions, quantifiers, `fold`, and structural
  recursion (`function recursive`, checked by a termination pass) are theorem-level
  terminating; general recursion is an escape hatch verified by bounded model checking
  (`with depth n`).
- **Verifiable**: built-in `test` blocks and a Stateright-based bounded model checker give
  per-property verdicts for `invariant`s and temporal `property`s.

Relationship to MuduDB: MuduDB is one of CQL's target backends — the plan is to compile CQL
into MuduDB WASI components that execute through query/command syscalls (see "Development
Status" below; that backend is currently a proposal-stage placeholder). The default backend
available today is Rust code generation.

## 2. Development Status

**Note: Churcuring is under active development and not all features are implemented yet.**
The following limitations apply to the current version:

- `use` supports only single-segment module names (`use util;`); cross-module query/action
  calls are not supported (compile-time error).
- The `mududb` backend is a **proposal-stage placeholder**: `cqlc build --backend mududb`
  only emits a deployment-plan text file and does not build a component; the syscall
  contract is still a proposal.
- Model checking covers a v1 fragment: bool/int expressions only, tables with int keys and
  a single int value field; prime (next-state), `~>`, and `until` are skipped with a
  warning by the Stateright backend; `fairness` declarations are accepted but no backend
  enforces them (warning only); `--replay` (counterexample replay) is unimplemented.
- The z3 model-checking backend is unavailable by default (requires the extra `z3` feature
  and a prebuilt Z3); `cqlc verify` exposes only the Stateright backend.
- IndexScan read plans currently compile to filtered full scans (result semantics
  unaffected — performance only).
- The tree-sitter wasm grammar package for the VSCode extension is not built yet; syntax
  highlighting degrades gracefully.

## 3. Syntax Overview

This section is a tour; normative details live in `doc/cn/cql.md` of the churcuring
repository (Chinese authoritative version).

### 3.1 Modules and Projects

Each `.cql` file is a module whose first line declares the module name. A project is
delimited by `cql.toml`; `cqlc` builds a dependency graph from `use` declarations and
compiles in topological order — cyclic dependencies are forbidden.

```text
module shop;

use util;                       // import all public items of module util

table users { id: int, name: string, city: string } primary key {id}

query large_orders() -> set<orders> == {
    read(orders, lambda(o) { is_large_amount(o.amount) })
}
```

Declarations marked `public` can be used across modules; **tables cannot cross modules**
(a schema and the queries/actions that access it directly must live in the same module).

### 3.2 Types

Basic types: `bool`, `int` (i64), `float` (f64), `string` (UTF-8), `decimal(m, n)`
arbitrary-precision fixed-point, `date`.

Container and composite types:

```text
option<int>                     // possibly absent; constructors some(x) / none
vector<int>                     // ordered sequence, [1, 2, 3]
set<string>                     // unordered, deduplicated; set {1, 2}
bag<float>                      // multiset; bag {1.0, 2.0, 2.0}
map<string, int>                // pure associative value; map { "a": 1 }
(int, string)                   // tuple; projection t.0 / t.1
{ id: int, name: string }       // record type
int -> int                      // pure function type (right-associative)
```

A table declaration `table users { ... }` automatically derives three types: `users` (the
full row type), `key users` (the key type), and `value users` (the non-key field record);
`lookup(users, k)` returns `option<value users>`. There are also `enum`s (variants may
carry payloads, be generic, and be recursive) and generic functions (explicit arguments use
turbofish: `f::<int>(x)`). No implicit conversions; `as` conversions follow a whitelist.

### 3.3 Declarations at a Glance

```text
const max_retries: int == 3;                          // compile-time constant
type user_id == int;                                   // type alias
table orders { order_id: int, user_id: int, amount: float }
    primary key {order_id}
    foreign key {user_id} references users             // table + key constraints
index sessions_by_user on sessions(user_id)            // secondary index

function is_adult(age: int) -> bool == { age >= 18 }   // pure function (L0)
query orders_by_user(user_id: int) -> set<orders> == { // query (L1, snapshot reads)
    read(orders, lambda [user_id](o) { o.user_id = user_id })
}
action add_user(id: int, name: string, city: string) -> set<write_op> == {  // action (L2)
    set { insert(users, record { id: id, name: name, city: city }) }
}

invariant non_negative on orders == \A o \in orders : o.amount >= 0.0
property balance_ok == [](total_balance() = 10000)     // temporal property

test transfer_basic {                                  // test block
    fixture accounts == [record { id: 1, owner: "a", balance: 6000 }];
    expect total_balance() == 6000;
}
```

Key points: definitions always use `==`, predicate equality uses `=`; operator bodies are
always blocks `{ ... }`.

### 3.4 Effect Tiers

| Tier | Construct | Allowed effects |
| --- | --- | --- |
| L0 | `function` | none (pure) |
| L1 | `query` | snapshot reads (`read`/`lookup`) |
| L2 | `action` | snapshot reads + produce `set<write_op>` (`insert`/`update`/`delete`) |

Tiers in the call graph may only stay level or rise: a `function` may only call
`function`s; a `query` may call `function`s/`query`s (sharing one snapshot); an `action`
may call all tiers (write_op sets of called actions merge in; atomicity only at the top
level). Calling downward is a compile error. Lambda bodies are always L0.

Three write_op constructors: `insert(t, row)` (key must not exist), `update(t, k, f)` (key
must exist; `f` is evaluated against the current row value at apply time), `delete(t, k)`
(no-op if the key does not exist). Apply order: conflict check → FK validation → invariant
validation; any violation **rejects the whole action** (no writes applied).

### 3.5 Expressions

```text
-- blocks and let (the block's value = its final expression)
{ let active == set { v \in users if v.active };
  set { u.name : u \in active } }

-- if / match (expressions; match exhaustiveness is statically checked)
if f.balance >= amt then set { ... } else set {}
match lookup(users, id) { some(v) => v.name, none => "unknown" }

-- set comprehension (two forms; result is set<T>, deduplicated)
set { x \in users if x.active }                        -- filter form
set { (o.order_id, u.name) : o \in orders, u \in lookup(users, o.user_id) }  -- map form

-- quantifiers
\A o \in orders : o.amount >= 0.0
\E u \in users : u.city = "x"

-- lambda: the capture list must explicitly name referenced outer locals
lambda [new_city](v) { record { v with city: new_city } }

-- ? propagation sugar: none makes the whole operator body none
{ let u == lookup(users, user_id)?; some(u.city) }
```

### 3.6 Termination

Two tiers:

```text
function recursive inorder(t: tree) -> vector<int> == {   -- structural recursion: termination pass proves termination
    match t {
        leaf(v)       => [v],
        node(l, x, r) => concat_vector(concat_vector(inorder(l), [x]), inorder(r))
    }
}

function gcd(a: int, b: int) -> int == {                  -- general recursion: free-form, may trap on stack exhaustion
    if b = 0 then a else gcd(b, a % b)
}
query subordinates(mgr_id: int) -> set<int> with depth 32 == { ... }  -- model-checking depth bound
```

## 4. Compilation and Toolchain

The Churcuring toolchain is a Rust workspace. Main components:

| Component | Role |
| --- | --- |
| `tree-sitter-cql` | tree-sitter grammar and parser |
| `cql-compiler` | compiler library: parse → name resolution → effect check → type check → termination → desugar → optimize → codegen |
| `cql-runtime` | runtime library: Value types, Table trait, in-memory tables, pure stdlib functions |
| `cql-mc` | model checker (Stateright / z3 backends) |
| `cql-cli` | the `cqlc` command-line tool |

### 4.1 Building the Toolchain

Prerequisite: Rust nightly 1.94. From the churcuring repository root:

```sh
cargo build --workspace --offline    # produces ./target/debug/cqlc
cargo test --workspace --offline     # full test suite
```

### 4.2 cqlc Subcommands

`cqlc <cmd> [path]` searches upward from `path` for `cql.toml` to enter project mode;
passing a `.cql` file directly selects single-file zero-config mode.

- `cqlc new <name>`: scaffolds `<name>/cql.toml` + `<name>/src/main.cql`.
- `cqlc check [path]`: runs the full pipeline and reports diagnostics only (rendered
  graphically via miette, with source snippets and spec section references).
- `cqlc build [path] [--backend rust|mududb]`: after a successful check, generates code
  per module in topological order into `out_dir`, then runs `cargo build --offline`. The
  `rust` backend (default) writes a standalone cargo crate; the `mududb` backend currently
  only emits a deployment-plan text file (proposal-stage placeholder).
- `cqlc test [path]`: generates the crate and runs `cargo test` (CQL `test` blocks compile
  to Rust `#[test]`s).
- `cqlc verify [path]`: model checking (see section 5).
- `cqlc clean [path]`: deletes `out_dir`.

### 4.3 cql.toml

```toml
[package]
name = "shop"            # required; also used as the generated crate's package name
version = "0.1.0"        # optional, default "0.1.0"

[build]
source_root = "src"        # source root, default "src"
out_dir = "target/cql"     # generated-code directory, default "target/cql"
backend = "rust"           # "rust" (default) | "mududb" (proposal-stage placeholder)

[mududb]                   # used when backend = "mududb" (draft)
app = "shop"
sql_adapter = "off"        # "off" (default) | "sqlite" | "postgres" | "mysql"
```

## 5. Testing and Verification

### 5.1 test Blocks

CQL source files may contain `test` blocks: `fixture` builds in-memory tables by primary
key, `expect` compares with predicate equality. `cqlc test` compiles them into Rust
`#[test]`s and runs `cargo test`:

```text
test transfer_basic {
    fixture accounts == [record { id: 1, owner: "a", balance: 6000 }];
    expect total_balance() == 6000;
}
```

```sh
./target/debug/cqlc test examples/bank_project
```

### 5.2 Model Checking (cqlc verify)

`cqlc verify` (project mode only) lowers the desugared AST to a model-checking spec (v1
fragment: bool/int expressions + tables with int keys and a single int value field) and
the Stateright explicit-state backend gives a verdict per property:

```text
$ ./target/debug/cqlc verify examples/bank_project
verifying `bank` (stateright): 1 table(s), 1 transition(s), 2 of 2 propert(ies), k=8
  PROVED(stateright-exhaustive) balance_conserved
  PROVED(stateright-exhaustive) no_negative
result: all 2 propert(ies) hold within the bounds
```

- Checking has two layers: bounded (Always: invariants/safety) and temporal (Eventually);
  select them with `--bounded` / `--temporal`; `--depth N` overrides recursion depth
  bounds; `--trace N` overrides the trace length bound (default k=8).
- Counterexamples come with a shortest path (BFS), each step rendering the action,
  arguments, result, and state diff.
- Discipline of claims: `PROVED` means proved **within the bounded model only**;
  prime/`~>`/`until` properties are skipped (warning, not counted in the verdict).

`verify.toml` (at the project root, next to `cql.toml`; all defaults if missing):

```toml
[depth]
default = 32          # default recursion depth bound
subordinates = 16     # per-operator override

[domain]
accounts.id = "1..2"              # key/field domain: range string
accounts.balance = [0, 6000, 4000]  # or an explicit value set (integers only)

[trace]
length = 8            # trace length bound k (default 8)
```

## 6. Example Projects

The churcuring repository's `examples/` directory contains compilable examples:

| Example | Contents |
| --- | --- |
| `examples/shop_project` | multi-file project (`util` + `shop` modules) with `cql.toml` |
| `examples/bank_project` | bank transfer example with `test` blocks, `invariant`s and `property`s; verifiable |
| `examples/analytics.cql` | single-file zero-config example |

Common smoke-test commands:

```sh
./target/debug/cqlc check examples/shop_project
./target/debug/cqlc test examples/bank_project
./target/debug/cqlc verify examples/bank_project
```

## 7. References

Specifications and guides inside the churcuring repository (`churcuring-p/`):

| Document | Contents |
| --- | --- |
| `doc/cn/cql.md` | CQL language specification (type system, modules, expressions, semantic rules) |
| `doc/cn/cql-tour.md` | CQL language tour (syntax walkthrough with compilable examples) |
| `doc/cn/cli.md` | full reference for `cqlc` subcommands, `cql.toml`, and `verify.toml` |
| `doc/cn/build.md` | build guide (environment, offline builds, platform gotchas) |
| `doc/cn/model-check.md` | formal model-checking mechanism (bounded/temporal layers, dual-backend architecture) |
| `doc/cn/backend-mududb.md` | mududb backend: query/command syscall contract (proposal) |

(The `doc/en/` directory contains English translations of these documents.)
