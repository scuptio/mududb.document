# 可挂起的过程执行

MuduDB 托管导入同步或异步宿主调用的组件。异步 world 使用 Wasmtime `call_async`：guest 等待异步宿主调用时可以挂起并返回 `Poll::Pending`，执行器可继续推进其他调用。一个 Store 同时最多执行一次调用。

## 每次调用使用全新状态

历史类型名 `ProcedureInstancePool` 现在表示实例工厂。每次 lease 都创建新的 `Store`、WASI 上下文和组件实例；仅复用已编译的组件代码和预实例化元数据。guest 线性内存、全局变量和句柄不会跨调用保留。调用方的 worker 上下文与调用预算在实例化前绑定。

Store 执行内存/表大小限制、fuel 和 epoch deadline；异步 Store 还按 fuel 间隔协作让出执行权。准入受全局/应用并发和内存预算限制，见[配置参考](../administration/configuration.md)。

## 同步调用

同步 guest 使用有界执行器：四个普通 worker 线程和有界任务队列。服务端通过 `call_on_executor` 异步等待结果，不会为每个请求无限制创建线程，也不会在数据库执行器上阻塞 join。若兼容路径调用导入异步接口的组件，则在执行器线程上的 current-thread runtime 中驱动它。

## 事务与取消生命周期

运行时建立任务/会话上下文，按需启动事务，执行组件，清理宿主资源，然后提交或回滚。提交前发生失败或预算耗尽会回滚该调用拥有的事务。一旦跨过提交边界，取消就不再改变结果。提交失败可能意味着结果未知，不能当作已确认回滚。

并发调用需要独立任务上下文。本地适配器和跨 worker 写入另有事务限制，见 [Mudu 过程](../programming/mudu_procedure.md)。

## 实现依据

- `crates/db-kernel/runtime_wasm/src/service/procedure_instance_pool.rs`
- `crates/db-kernel/runtime_wasm/src/service/procedure_invoke_component.rs`
- `crates/db-kernel/runtime_wasm/src/service/invocation_executor.rs`
- `crates/db-kernel/runtime_wasm/src/service/app_inst_impl.rs`
