# SDK 与执行模式

MuduDB 提供 Rust、AssemblyScript、Python、C#、Go 和 C 六种语言绑定。使用时需明确选择执行模式：

| 模式 | 业务逻辑执行位置 | 要求 |
|------|------------------|------|
| MPK guest | `mudud` 托管的 WebAssembly 组件中 | 构建并安装 `.mpk`，通过 `mcli app-invoke` 或客户端调用。 |
| 本地适配器 | 本地应用进程中 | 通过语言 API 选择 adapter，并加载单独构建的本地运行库。 |

`MUDU_CONNECTION` 选择适配器的数据库后端，不会将本地应用切换为服务端过程。本地模式支持 SQLite、PostgreSQL、MySQL 和远程 Mudud，SQL 能力与事务行为取决于后端。Wallet 示例用于演示调用，不代表生产账务系统的并发保证。

## 各语言入口

下列路径相对于 MuduDB 源码仓库。

| 语言 | MPK 示例 | 本地适配器入口 |
|------|----------|----------------|
| AssemblyScript | `example/mpk/wallet-as` | `crates/sdk/bindings/assemblyscript/adapter/`（Node.js） |
| Python | `example/mpk/wallet-py` | `mududb.adapter` |
| C# | `example/mpk/wallet-cs` | `NativeAdapter` |
| Go | `example/mpk/wallet-go` | `github.com/mududb/mududb-go/adapter` |
| C | `example/mpk/wallet-c` | `mududb/adapter/` 头文件 |
| Rust | `example/mpk/wallet` | 独立 SDK 的 `mudu_api_rust::adapter` |

仓库内的 Rust `mududb` facade 与导出的 `mudu_api_rust` SDK 是不同的包；消费者可以在 Cargo 中将后者重命名为 `mududb`。C 与 Go 的类型/编解码绑定本身不是完整的 guest 数据库 facade，MPK 示例还提供宿主调用桥接。各语言 API 覆盖范围并不完全相同。

## 构建 SDK 分发包

在 MuduDB 仓库根目录执行：

```bash
cargo make build-sdk
SDK_LANGS=py,go cargo make build-sdk
cargo make build-sdk --with-adapter-runtime
SDK_LANGS=py,go cargo make build-sdk --with-examples
cargo make build-sdk --with-examples --example-mode adapter
```

默认构建全部六种 SDK，需要安装所选语言的全部工具链。公共依赖为 Python 3.11+、Git、仓库固定的 Rust 工具链、cargo-make 和本地 C/C++ 编译器。语言依赖包括 Node.js 22/npm（AS）、Python `build`（Python）、.NET 8 SDK 与运行时（C# SDK 测试）、Go 1.24+（Go）、Clang/Clang++（C）及 `wasm32-wasip2`（Rust）。

默认输出为 `target/sdk-dist/`；可用 `SDK_OUTPUT_DIR`、`SDK_PYTHON` 和 `CARGO_TARGET_DIR` 指定输出目录、解释器与 Cargo 构建目录。每种语言包含 `repo/` 与 `artifacts/`；整批成功后写入含源码版本和验证结果的 `manifest.json`。该命令只构建分发包，发布使用独立的手动工作流。

每个导出 SDK 都包含 `examples/wallet/mpk/` 和 `examples/wallet/adapter/`。在导出的 Wallet 目录执行 `mpm build`，读取 `mpm.toml` 并生成 `target/wallet-<language>.mpk`；这与仓库内 Rust Wallet 的 `cargo make package` 路径不同。`mpm build --manifest-path PATH --python PYTHON` 可指定清单与解释器。

## 示例的额外工具链

| 组件 | 工具要求 |
|------|----------|
| 本地适配器运行库 | Rust、本地 C/C++、CMake、Node.js、Clang |
| MPK guest | `wasm-tools` 1.252.0 |
| Python guest | `componentize-py` 0.25.1 |
| Go guest | TinyGo 0.42.0 与兼容的 Go |
| C guest | wasi-sdk 29，通过 `WASI_SDK` 选择 |
| C# guest | .NET 10；完整 guest 测试矩阵目前面向 Linux x64 |
| C# 本地适配器 | .NET 8 |
| Go 本地适配器 | cgo 与 C 编译器 |

解压匹配宿主平台的 adapter runtime 包，将 `MUDU_ADAPTER_LIBRARY` 设置为运行库的绝对路径。AS 还需将 `mudu_adapter.node` 放在运行库旁。普通 SDK 源码导出不包含本地运行库二进制。进程启动前设置连接，例如：

```bash
export MUDU_CONNECTION='sqlite://wallet.db'
# 本地应用连接到已有的 Mudud 应用/schema：
export MUDU_CONNECTION='mudud://127.0.0.1:9527/wallet-py?http_addr=127.0.0.1:8300'
```

Mudud Wallet adapter 需先构建并安装 MPK 以建立 schema，再跳过 adapter 的 `init`，运行 `scenario`。验证不同执行模式时应使用各自独立的数据库。

实现和完整构建要求见 MuduDB 仓库中的 `doc/dev/sdk_distribution.md`、`script/sdk/config.json`、`script/sdk/sdk_build.py` 及各语言示例。
