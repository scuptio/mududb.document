# Toolchain

The MuduDB toolchain is organized around building, packaging, deploying, and invoking procedure applications.

## Tool overview

| Tool | Purpose | Typical usage |
|------|---------|---------------|
| `mudud` | The MuduDB server process. | `mudud` or `mudud serve --cfg ./mudud.cfg` |
| `mcli` | TCP client, interactive SQL shell, and HTTP management CLI. | `mcli shell`, `mcli app-install`, `mcli server-topology` |
| `mgen` | Source generator. Creates typed Rust entities from SQL DDL. | `mgen entity -i ddl.sql -o ./src/generated -t type.desc.json -l rust` |
| `mtp` | Transpiler. Rewrites synchronous procedure source into async-compatible form for the WASM runtime. | `mtp rust -i src/procedures.rs -o src/generated/procedures.rs --async` |
| `mpm` | Project builder. Runs the Python build script declared in `mpm.toml`. | `mpm build` |
| `mpm-build` | Package builder. Produces `.mpk` archives from DDL, descriptors, and WASM modules. | `mpm-build create --package-cfg package.cfg.json --package-desc package.desc.json --ddl-sql ddl.sql --initdb-sql initdb.sql --wasm-files wallet.wasm` |
| `mpm-crate` | Project scaffolder. Creates a new `.mpk` application project from a template (like `cargo new`). | `mpm-crate my-app --lang rust` |
| `mpm-install` | MPK installer helper (also installable through `mcli app-install`). | `mpm-install wallet.mpk` |
| `mudup` | Release installer. Downloads released binaries without building from source. | `mudup install` — **planned (design proposal); not available yet — build from source.** |

## Typical workflow

1. **Define schema**: write `ddl.sql` for your application tables.
2. **Generate types**: run `mgen` to produce typed `Entity` structs from the DDL.
3. **Write procedures**: implement functions marked with `/**mudu-proc**/`, using `sys_interface::sync_api`.
4. **Transpile**: run `mtp` to convert blocking syscalls into async form for the component runtime.
5. **Build WASM**: compile the transpiled source to a `wasm32-wasip2` component that imports the host syscall interface and exports the procedure entry points.
6. **Package**: run `mpm-build create` to produce a `.mpk` file.
7. **Install**: run `mcli app-install --mpk <file>.mpk` on the target server.
8. **Invoke**: run `mcli app-invoke --app <app> --module <module> --proc <proc> --json '{...}'`.

See [Application Packages](packages.md) for the package format and [Your First MPK Package](../tutorial/tutorials/wallet.md) for a complete walkthrough.
