# 概览

MuduDB 围绕数据本地业务过程执行而构建。

数据库内核拥有存储、事务、查询执行和调度控制权。用户过程以 WebAssembly Component Model 模块的形式托管，并通过受控的系统调用访问数据库能力。

这种结构减少了关键路径上的跨进程和跨网络流量，同时将正确性边界集中在数据库内核中。

## 术语

| 术语 | 含义 |
|------|------|
| **Mudu 过程（MP）** | 在 MuduDB 引擎内部、靠近数据运行的用户定义函数。 |
| **OID** | 对象标识符；作为每个过程第一个参数传入的会话/事务句柄。 |
| **MPK** | MuduDB 包；包含 DDL、描述符与编译后 WebAssembly 组件的 ZIP 归档。 |
| **App / Module / Procedure** | 安装后的包层次结构：应用包含模块，模块包含过程。 |
| **Kernel（内核）** | 数据库正确性基础：存储、事务、查询执行、调度。 |
| **Runtime（运行时）** | 在内核控制下运行用户过程的 WebAssembly 组件宿主。 |
| **Partition（分区）** | 通过分区规则与 placement 元数据映射到工作线程的数据逻辑分片。 |
| **Worker（工作线程）** | 拥有部分分区与本地状态的每核执行线程。 |
| **Primary / Replica（主/从）** | 流复制中，主库接受写并推送 WAL 段；从库只读并重放。 |
| **Raft** | 基于共识的容错：节点组选举 leader、多数派提交、故障自愈。 |
| **Epoch（纪元）** | 单调递增的代数，用于人工故障切换时隔离旧主。 |

## 下一步

- [数据模型](data-model.md)
- [入门](../tutorial/getting-started/index.md)
- [Mudu 过程](../programming/mudu_procedure.md)
- [内核](../internal/kernel.md)
- [参考资料](../reference/index.md)
