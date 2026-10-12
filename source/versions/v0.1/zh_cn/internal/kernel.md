# 内核

MuduDB 内核是系统的正确性基础。它负责存储、事务处理、查询执行、调度，以及过程访问数据库状态时使用的系统调用边界。

## 内核与运行时分层

MuduDB 采用双层架构：

- **内核（Kernel）** —— 提供数据库引擎：页、关系表、索引、WAL、事务、查询计划/执行与调度。
- **运行时（Runtime）** —— 托管用户定义的过程（WebAssembly 组件），并将其系统调用转发到内核。

运行时是被动的：它不引入自己的调度器或执行策略。过程执行由内核触发和控制，以确保调度、正确性与数据访问都处在同一权威之下。

## 职责

| 领域 | 内核职责 |
|------|----------|
| 存储 | 管理关系表与时序文件、页分配、slot 数组及磁盘格式。 |
| 事务 | 提供会话上下文、冲突检测、提交/回滚与 WAL 持久化。 |
| 查询执行 | 解析并执行 SQL 查询、执行分区路由、返回结果集。 |
| 调度 | 将工作分配给每核工作线程，并协调基于续体的异步执行。 |
| 系统调用 | 暴露窄系统调用接口（`mudu_open`、`mudu_query`、`mudu_get` 等）。 |
| 兼容性 | 跟踪格式版本，并通过升级/回滚处理器路由持久化数据。 |

## 工作线程模型

内核围绕每核工作线程组织执行。每个工作线程拥有部分分区、本地连接状态，以及（在 `IOUring`/`Tokio` 模式下）本地异步运行时。这样可将热路径数据保留在同一核心，减少跨核争用。

更多细节请参见[每核工作线程模型](per_core_worker.md)与[分区](partitioning.md)。

## 边界与兼容性

内核的磁盘格式与网络线格式由[参考资料](../reference/contracts/index.md)中的版本化契约管理。面向用户的文档应描述子系统边界，而不应将实现细节转化为兼容性承诺。

## 源码布局

仓库根目录没有 `Cargo.toml`。`crates/common`、`crates/db-kernel`、`crates/sdk`、`crates/tools` 是独立工作区；`example/mpk`、`crates/sdk/mudu_api/rust` 和 `crates/bindings` 也有独立构建上下文。本地适配器 ABI 工作区使用隔离的 target 目录。

`mudu_kernel` 是十四个内核 crate 的 facade：`catalog`、`index`、`wal`、`xengine`、`storage`、`meta`、`command`、`sql`、`conn`、`xcontract`、`worker`、`repl`、`backup`、`server`。`runtime_wasm` 托管组件，`mudu_runtime` 集成协议服务和管理入口，`mudu_raft` 实现共识。依赖规则见源码仓库的 `ARCHITECTURE.md`。

应按工作区执行检查，例如 `cargo test --manifest-path crates/db-kernel/Cargo.toml --workspace`。根级协调命令为 `cargo make`、`cargo make check`、`cargo make test`。
