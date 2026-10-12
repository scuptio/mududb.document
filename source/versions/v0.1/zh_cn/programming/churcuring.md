# Churcuring 与 CQL

```{note}
本章描述外部语言项目及设计资料，不属于 MuduDB 已实现的安装流程或 API 契约。当前 MuduDB 工作区不提供 `cqlc`，可运行示例应使用本文档的 MPK/SDK 路径。本次文档更新未重新验证外部编译器的开发状态。
```


## 1. 概述

Churcuring 是一个面向 MuduDB 的声明式编程工具链，其核心语言为 **CQL（Churcuring Query
Language）**——一种声明式、强类型的查询与业务逻辑语言。CQL 借鉴 TLA+ 的集合推导、量词
与定义结构，采用 Rust 风格命名，目标是让形式化规约可以直接编译为可执行、可验证的
数据库逻辑。

主要设计要点：

- **不使用 JOIN**：一切跨表关联都通过显式 `lookup` 完成（`table = { Key → Object }`），
  JOIN 在语言和编译器层面被消除，而非仅仅不推荐。
- **效应分层**：所有计算都是纯的；唯一的副作用是表读（`read`/`lookup`）与表写
  （`insert`/`update`/`delete`），并组织为三个算子层级（L0 `function` / L1 `query` /
  L2 `action`），调用只能向高层级组合。
- **完全确定性**：相同快照 + 相同参数 ⇒ 位级一致的结果，包括一致的 trap 行为。
- **默认可证终止**：推导、量词、`fold` 与结构递归（`function recursive`，由终止性
  pass 检查）在定理层面终止；一般递归作为逃生舱，由有界模型检查验证
  （`with depth n`）。
- **可验证**：内置 `test` 块与基于 Stateright 的有界模型检查器，可对 `invariant`
  与时态 `property` 给出逐条结论。

CQL 与 MuduDB 的关系：MuduDB 是 CQL 的目标后端之一——规划中将 CQL 编译为 MuduDB 的
WASI 组件，通过 query/command 系统调用执行（见下文「开发状态」，该后端当前为提案态
占位）。当前可用的默认后端是 Rust 代码生成。

## 2. 开发状态

**注意：Churcuring 正在开发中，功能尚未完全实现。** 以下限制适用于当前版本：

- `use` 仅支持单段模块名（`use util;`）；跨模块 query/action 调用暂不支持
  （编译期报错）。
- `mududb` 后端为**提案态占位**：`cqlc build --backend mududb` 只生成部署计划文本，
  不做组件构建；syscall 契约仍是提案。
- 模型检查为 v1 片段：仅支持 bool/int 表达式、int 键与单 int 值字段的表；
  prime（次态）、`~>`、`until` 在 Stateright 后端被跳过并告警；`fairness` 声明被
  接受但无后端强制（仅告警）；`--replay`（反例重放）未实现。
- z3 模型检查后端默认不可用（需要额外的 `z3` feature 与预编译 Z3）；`cqlc verify`
  只暴露 Stateright 后端。
- IndexScan 读计划当前编译为带过滤的全扫（结果语义不受影响，仅性能）。
- VSCode 插件的 tree-sitter wasm 语法包尚未构建，高亮功能降级。

## 3. 语法概览

本节为导览性质；规范性细节以 churcuring 仓库的 `doc/cn/cql.md` 为准。

### 3.1 模块与工程

每个 `.cql` 文件是一个模块，首行声明模块名；工程由 `cql.toml` 界定，`cqlc` 按
`use` 建立依赖图、以拓扑序编译，禁止循环依赖。

```text
module shop;

use util;                       // 导入 util 模块的全部 public 项

table users { id: int, name: string, city: string } primary key {id}

query large_orders() -> set<orders> == {
    read(orders, lambda(o) { is_large_amount(o.amount) })
}
```

`public` 导出的声明可跨模块使用；**表不可跨模块**（schema 与直接访问它的
query/action 须放在同一模块）。

### 3.2 类型

基础类型：`bool`、`int`（i64）、`float`（f64）、`string`（UTF-8）、
`decimal(m, n)` 任意精度定点、`date`。

容器与复合类型：

```text
option<int>                     // 可能缺失；构造子 some(x) / none
vector<int>                     // 有序序列，[1, 2, 3]
set<string>                     // 无序去重；set {1, 2}
bag<float>                      // 多重集；bag {1.0, 2.0, 2.0}
map<string, int>                // 纯关联值；map { "a": 1 }
(int, string)                   // 元组；投影 t.0 / t.1
{ id: int, name: string }       // 记录类型
int -> int                      // 纯函数类型（右结合）
```

表声明 `table users { ... }` 自动派生三个类型：`users`（全字段行类型）、
`key users`（键类型）、`value users`（非键字段记录）；`lookup(users, k)` 返回
`option<value users>`。另有 `enum`（变体可带载荷、可泛型、可递归）与泛型函数
（显式实参用 turbofish：`f::<int>(x)`）。无隐式转换，`as` 转换有白名单。

### 3.3 声明速览

```text
const max_retries: int == 3;                          // 编译期常量
type user_id == int;                                   // 类型别名
table orders { order_id: int, user_id: int, amount: float }
    primary key {order_id}
    foreign key {user_id} references users             // 表 + 键约束
index sessions_by_user on sessions(user_id)            // 二级索引

function is_adult(age: int) -> bool == { age >= 18 }   // 纯函数（L0）
query orders_by_user(user_id: int) -> set<orders> == { // 查询（L1，读快照）
    read(orders, lambda [user_id](o) { o.user_id = user_id })
}
action add_user(id: int, name: string, city: string) -> set<write_op> == {  // 动作（L2）
    set { insert(users, record { id: id, name: name, city: city }) }
}

invariant non_negative on orders == \A o \in orders : o.amount >= 0.0
property balance_ok == [](total_balance() = 10000)     // 时态性质

test transfer_basic {                                  // 测试块
    fixture accounts == [record { id: 1, owner: "a", balance: 6000 }];
    expect total_balance() == 6000;
}
```

要点：定义一律用 `==`，谓词相等用 `=`；算子体一律为块 `{ ... }`。

### 3.4 效应层级

| 层级 | 构造 | 允许的效应 |
| --- | --- | --- |
| L0 | `function` | 无（纯） |
| L1 | `query` | 读快照（`read`/`lookup`） |
| L2 | `action` | 读快照 + 产出 `set<write_op>`（`insert`/`update`/`delete`） |

调用图中层级只能持平或上升：`function` 只能调 `function`；`query` 可调
`function`/`query`（共享同一快照）；`action` 可调用全部层级（被调 action 的
write_op 集合并入，原子性只在顶层）。反向调用是编译错误。lambda 体恒为 L0。

写操作三构造：`insert(t, row)`（键必须不存在）、`update(t, k, f)`（键必须存在，
`f` 对应用时的当前行值求值）、`delete(t, k)`（键不存在为 no-op）。应用顺序为：
冲突检查 → 外键校验 → invariant 校验，任一违反则**拒绝整个 action**（不应用任何写）。

### 3.5 表达式

```text
-- 块与 let（块的值 = 末位表达式）
{ let active == set { v \in users if v.active };
  set { u.name : u \in active } }

-- if / match（表达式；match 穷尽性静态检查）
if f.balance >= amt then set { ... } else set {}
match lookup(users, id) { some(v) => v.name, none => "unknown" }

-- 集合推导（两形式，结果为 set<T> 去重）
set { x \in users if x.active }                        -- 过滤式
set { (o.order_id, u.name) : o \in orders, u \in lookup(users, o.user_id) }  -- 映射式

-- 量词
\A o \in orders : o.amount >= 0.0
\E u \in users : u.city = "x"

-- lambda：捕获列表必须显式列出引用的外层局部绑定
lambda [new_city](v) { record { v with city: new_city } }

-- ? 传播糖：none 则整个算子体为 none
{ let u == lookup(users, user_id)?; some(u.city) }
```

### 3.6 终止性

双层制：

```text
function recursive inorder(t: tree) -> vector<int> == {   -- 结构递归：termination pass 证明终止
    match t {
        leaf(v)       => [v],
        node(l, x, r) => concat_vector(concat_vector(inorder(l), [x]), inorder(r))
    }
}

function gcd(a: int, b: int) -> int == {                  -- 一般递归：写法自由，运行时可栈溢出 trap
    if b = 0 then a else gcd(b, a % b)
}
query subordinates(mgr_id: int) -> set<int> with depth 32 == { ... }  -- 模型检查深度界
```

## 4. 编译与工具链

Churcuring 工具链是一个 Rust workspace，主要组件：

| 组件 | 作用 |
| --- | --- |
| `tree-sitter-cql` | tree-sitter 语法与解析器 |
| `cql-compiler` | 编译库：解析 → 名字解析 → 效应检查 → 类型检查 → 终止性 → 脱糖 → 优化 → 代码生成 |
| `cql-runtime` | 运行时库：Value 类型、Table trait、内存表、标准库纯函数 |
| `cql-mc` | 模型检查器（Stateright / z3 后端） |
| `cql-cli` | 命令行工具 `cqlc` |

### 4.1 构建工具链

前置条件：Rust nightly 1.94。在 churcuring 仓库根目录：

```sh
cargo build --workspace --offline    # 产物 ./target/debug/cqlc
cargo test --workspace --offline     # 全量测试
```

### 4.2 cqlc 子命令

`cqlc <cmd> [path]` 从 `path` 向上查找 `cql.toml` 进入工程模式；直接给 `.cql` 文件
则为单文件零配置模式。

- `cqlc new <name>`：脚手架，生成 `<name>/cql.toml` + `<name>/src/main.cql`。
- `cqlc check [path]`：跑完整编译流水线，只报诊断（miette 图形化渲染，含源码
  片段与规范章节引用）。
- `cqlc build [path] [--backend rust|mududb]`：check 通过后按拓扑序逐模块生成代码
  到 `out_dir` 再执行 `cargo build --offline`。`rust`（默认）后端写出一个独立
  cargo crate；`mududb` 后端当前只生成部署计划文本（提案态占位）。
- `cqlc test [path]`：生成 crate 后执行 `cargo test`（CQL `test` 块编译为 Rust
  `#[test]`）。
- `cqlc verify [path]`：模型检查（见第 5 节）。
- `cqlc clean [path]`：删除 `out_dir`。

### 4.3 cql.toml

```toml
[package]
name = "shop"            # 必填；也用作生成 crate 的包名
version = "0.1.0"        # 可选，默认 "0.1.0"

[build]
source_root = "src"        # 源码根目录，默认 "src"
out_dir = "target/cql"     # 生成代码目录，默认 "target/cql"
backend = "rust"           # "rust"（默认）| "mududb"（提案态占位）

[mududb]                   # backend = "mududb" 时使用（草案）
app = "shop"
sql_adapter = "off"        # "off"（默认）| "sqlite" | "postgres" | "mysql"
```

## 5. 测试与验证

### 5.1 test 块

CQL 源文件中可写 `test` 块：`fixture` 按主键构造内存表，`expect` 用谓词相等比较。
`cqlc test` 将其编译为 Rust `#[test]` 并执行 `cargo test`：

```text
test transfer_basic {
    fixture accounts == [record { id: 1, owner: "a", balance: 6000 }];
    expect total_balance() == 6000;
}
```

```sh
./target/debug/cqlc test examples/bank_project
```

### 5.2 模型检查（cqlc verify）

`cqlc verify`（仅工程模式）将脱糖后的 AST 降到模型检查规约（v1 片段：bool/int
表达式 + int 键、单 int 值字段的表），由 Stateright 显式状态后端逐条性质给出结论：

```text
$ ./target/debug/cqlc verify examples/bank_project
verifying `bank` (stateright): 1 table(s), 1 transition(s), 2 of 2 propert(ies), k=8
  PROVED(stateright-exhaustive) balance_conserved
  PROVED(stateright-exhaustive) no_negative
result: all 2 propert(ies) hold within the bounds
```

- 检查分两层：有界层（Always：invariant/安全性）与时态层（Eventually）；可用
  `--bounded` / `--temporal` 分别选择，`--depth N` 覆盖递归深度界，`--trace N`
  覆盖迹长度界（默认 k=8）。
- 反例附最短路径（BFS），每步渲染 action、参数、结果与状态差。
- 表述纪律：`PROVED` 仅指**有界模型内**的证明；prime/`~>`/`until` 性质被跳过
  （告警，不计入结论）。

`verify.toml`（放在工程根，与 `cql.toml` 同级；缺失则全部默认）：

```toml
[depth]
default = 32          # 递归深度界默认
subordinates = 16     # 按算子名覆盖

[domain]
accounts.id = "1..2"              # 键/字段域：区间串
accounts.balance = [0, 6000, 4000]  # 或显式值集合（仅整数值）

[trace]
length = 8            # 迹长度界 k（默认 8）
```

## 6. 示例工程

churcuring 仓库的 `examples/` 目录包含可编译示例：

| 示例 | 内容 |
| --- | --- |
| `examples/shop_project` | 多文件工程（`util` + `shop` 模块），带 `cql.toml` |
| `examples/bank_project` | 银行转账示例，含 `test` 块、`invariant` 与 `property`，可 `verify` |
| `examples/analytics.cql` | 单文件零配置示例 |

常用冒烟命令：

```sh
./target/debug/cqlc check examples/shop_project
./target/debug/cqlc test examples/bank_project
./target/debug/cqlc verify examples/bank_project
```

## 7. 参考

churcuring 仓库（`churcuring-p/`）内的规范与指南（`doc/cn/` 为中文权威版）：

| 文档 | 内容 |
| --- | --- |
| `doc/cn/cql.md` | CQL 语言规范（类型系统、模块、表达式、语义规则） |
| `doc/cn/cql-tour.md` | CQL 语言教程（语法导览，带可编译小例子） |
| `doc/cn/cli.md` | `cqlc` 子命令、`cql.toml` 与 `verify.toml` 完整参考 |
| `doc/cn/build.md` | 构建指南（环境、离线构建、平台已知问题） |
| `doc/cn/model-check.md` | 形式化模型检查机制（有界层/时态层、双后端架构） |
| `doc/cn/backend-mududb.md` | mududb 后端：query/command syscall 契约（提案） |
