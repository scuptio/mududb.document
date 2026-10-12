# WebAssembly 集成

MuduDB 在 WASI 沙箱中运行用户过程 WebAssembly 组件。它在保持数据库访问受内核控制的同时，提供可移植性、隔离性与多语言支持。

## 组件模型

MuduDB 过程被编译为面向 `wasm32-wasip2` 平台的 WebAssembly 组件。该组件导入少量宿主编译的系统调用，并导出标记为 `/**mudu-proc**/` 的过程函数。

构建时，`mtp` 转译器生成组件的导入/导出接口：组件导入宿主系统调用接口（同步 world 为 `mududb:api/system`，异步 world 为 `mududb:async-api/system`），并为每个过程导出一个 `mp2-<name>` 入口点，与运行时加载器期望的签名相匹配。

## WASI 沙箱

过程在 WASI 沙箱中执行，具有以下特性：

- 数据库访问只能通过显式内核系统调用进行。
- 内核中介所有网络、文件与存储访问。
- 可按每次过程调用强制执行资源限制（内存、CPU、系统调用次数）。

## 语言支持

运行时托管六种客户语言：Rust（主要过程语言）、AssemblyScript、C#、Python、C 和 Go。原则上，任何能生成 WASI 组件的语言未来都可支持。

| 语言 | 目标 | 说明 |
|------|------|------|
| Rust | `wasm32-wasip2` | 主要支持语言；通过 `sys_interface` 与 `mududb` SDK 提供绑定。 |
| AssemblyScript | WASI 组件 | 通过 `mtp assembly-script` 转译器前端与 shim 支持。 |
| C# | WASI 组件 | 使用 componentize-dotnet（.NET wasi-wasm 目标）构建。 |
| Python | WASI 组件 | 使用 componentize-py 构建；`mtp python` 前端。 |
| C | WASI 组件 | 使用 wasi-sdk 的 clang 工具链构建。 |
| Go | WASI 组件 | 使用 TinyGo 构建。 |

## 客户绑定

客户绑定由 `mgen` 根据 `mudu_binding/wit/` 中的 WIT 接口定义生成，该目录是客户/宿主 ABI 的唯一事实来源。仓库内的 Rust 过程通过 `sys_interface` 与 `mududb` SDK 门面使用这些绑定；每种客户语言各自携带生成的编解码器。

## 运行时限制

运行时可对每次过程调用强制执行以下限制：

- 内存使用；
- CPU 预算（fuel）与墙上时钟超时；
- 宿主调用（系统调用）次数；
- 输入与输出大小；
- 并发调用数（全局及每个应用）。

这些限制有助于将用户代码与内核及其他过程隔离。

## 包格式

编译后的组件与 DDL、描述符及可选初始化数据一起打包为 `.mpk` 归档。详见[应用包](../administration/packages.md)与 [MPK 清单契约](../reference/contracts/mpk_manifest_v1.md)。
