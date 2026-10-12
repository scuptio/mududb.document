# SDKs and Execution Modes

MuduDB has bindings for Rust, AssemblyScript, Python, C#, Go and C. Choose the execution mode explicitly:

| Mode | Where business logic runs | Requirements |
|------|---------------------------|--------------|
| MPK guest | A WebAssembly component hosted by `mudud` | Build and install an `.mpk`; invoke with `mcli app-invoke` or a client. |
| Native adapter | In the local application process | Select the adapter in the language API and load the separately built native runtime. |

`MUDU_CONNECTION` selects the adapter's database backend; it does not turn a local application into a server-side procedure. Native mode supports SQLite, PostgreSQL, MySQL and remote Mudud, with backend-specific SQL and transaction behavior. The Wallet examples demonstrate calls, not production accounting guarantees.

## Language entry points

Paths below are relative to the MuduDB source repository.

| Language | MPK example | Native adapter entry |
|----------|-------------|----------------------|
| AssemblyScript | `example/mpk/wallet-as` | `crates/sdk/bindings/assemblyscript/adapter/` (Node.js) |
| Python | `example/mpk/wallet-py` | `mududb.adapter` |
| C# | `example/mpk/wallet-cs` | `NativeAdapter` |
| Go | `example/mpk/wallet-go` | `github.com/mududb/mududb-go/adapter` |
| C | `example/mpk/wallet-c` | `mududb/adapter/` headers |
| Rust | `example/mpk/wallet` | `mudu_api_rust::adapter` in the standalone SDK |

The in-repository Rust `mududb` facade and the exported `mudu_api_rust` SDK are separate packages. A consumer may rename the latter to `mududb` in Cargo. C and Go type/codec bindings alone are not a complete guest database facade: their MPK examples also supply host-call bridges. API coverage is not identical across languages.

## Build SDK distributions

Run in the MuduDB repository root:

```bash
cargo make build-sdk
SDK_LANGS=py,go cargo make build-sdk
cargo make build-sdk --with-adapter-runtime
SDK_LANGS=py,go cargo make build-sdk --with-examples
cargo make build-sdk --with-examples --example-mode adapter
```

The default builds all six SDKs and requires all selected toolchains. Common requirements are Python 3.11+, Git, the pinned Rust toolchain, cargo-make and a native C/C++ compiler. Language requirements include Node.js 22/npm (AS), Python `build` (Python), .NET 8 SDK and runtime (C# SDK tests), Go 1.24+ (Go), Clang/Clang++ (C), and `wasm32-wasip2` (Rust).

Output defaults to `target/sdk-dist/`; `SDK_OUTPUT_DIR`, `SDK_PYTHON` and `CARGO_TARGET_DIR` can override locations and interpreter. Each language has `repo/` and `artifacts/`; a successful batch writes `manifest.json` with source revision and validation results. This command builds distributions; publication is a separate manual workflow.

Every exported SDK includes Wallet sources under `examples/wallet/mpk/` and `examples/wallet/adapter/`. In that Wallet directory, `mpm build` reads `mpm.toml` and produces `target/wallet-<language>.mpk`. This differs from the in-repository Rust Wallet's `cargo make package` path. `mpm build --manifest-path PATH --python PYTHON` selects a manifest and interpreter.

## Extra example toolchains

| Component | Required tools |
|-----------|----------------|
| Native adapter runtime | Rust, native C/C++, CMake, Node.js, Clang |
| MPK guests | `wasm-tools` 1.252.0 |
| Python guest | `componentize-py` 0.25.1 |
| Go guest | TinyGo 0.42.0 and compatible Go |
| C guest | wasi-sdk 29, selected with `WASI_SDK` |
| C# guest | .NET 10; the complete guest matrix currently targets Linux x64 |
| C# native adapter | .NET 8 |
| Go native adapter | cgo and a C compiler |

Extract the adapter runtime archive for your host and set `MUDU_ADAPTER_LIBRARY` to the library's absolute path. For AS keep `mudu_adapter.node` beside it. Ordinary SDK source exports contain no native runtime binary. Set the connection before process startup, for example:

```bash
export MUDU_CONNECTION='sqlite://wallet.db'
# Or connect the local application to an existing Mudud application/schema:
export MUDU_CONNECTION='mudud://127.0.0.1:9527/wallet-py?http_addr=127.0.0.1:8300'
```

For the Mudud Wallet adapter, first build/install its MPK to create the schema, then run the adapter's `scenario` without its `init` step. Use separate databases when validating each execution mode.

Implementation and full build requirements: `doc/dev/sdk_distribution.md`, `script/sdk/config.json`, `script/sdk/sdk_build.py` and the per-language examples in the MuduDB repository.
