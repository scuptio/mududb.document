# 数据类型

MuduDB 将值存储为类型化对象。SQL 解析器将 SQL 类型名称映射到内部标量种类。每种类型具有固定或可变的内存表示，以及定义良好的比较和序列化行为。

## 概述

类型分为标量和复杂类别：

- **标量** — 单一原子值（数字、字符串、时间值）。
- **复杂** — `ARRAY`、`RECORD` 和 `BINARY`。

所有标量类型支持排序和相等比较。复杂类型具有结构性比较规则。

下表总结了解析器识别的 SQL 类型名称及其内部映射。

| SQL 类型 | 内部种类 | 说明 |
|---|---|---|
| `INT` | `I32` | 32 位有符号整数 |
| `BIGINT` | `I64` | 64 位有符号整数 |
| `HUGEINT` | `I128` | 128 位有符号整数 |
| `FLOAT` | `F32` | 32 位 IEEE-754 浮点数 |
| `DOUBLE`, `REAL` | `F64` | 64 位 IEEE-754 浮点数 |
| `NUMERIC(p,s)` | `Numeric` | 定点十进制数（精度最大 38 位） |
| `DECIMAL(p,s)` | `Numeric` | `NUMERIC` 的别名 |
| `CHAR(n)` | `String` | 带声明长度的字符串（不填充、不强制截断） |
| `VARCHAR(n)` | `String` | 变长字符串 |
| `TEXT` | `String` | 无界文本 |
| `DATE` | `Date` | 日历日期 |
| `TIME(p)` | `Time` | 一天中的时间，可选小数秒精度 |
| `TIMESTAMP(p)` | `Timestamp` | 不带时区的日期+时间 |
| `TIMESTAMPTZ(p)` | `TimestampTz` | 带时区的日期+时间 |

```{toctree}
:maxdepth: 1

integer
floating-point
fixed-point
string
temporal
other
```

## 可空性

默认情况下，除非声明为 `NOT NULL` 或标记为 `PRIMARY KEY` 的一部分，否则每列都是可空的。`NULL` 是 `INSERT` 和 `UPDATE` 语句中的有效字面量。

```sql
CREATE TABLE example (
    id   INT PRIMARY KEY,        -- 隐式 NOT NULL
    name CHAR(32) NOT NULL,      -- 显式 NOT NULL
    note VARCHAR(200)            -- 默认可空
);
```

## 不支持的类型

以下类型已被语法识别，但尚未在 SQL 解析器中实现：

- `BOOLEAN`
- `SMALLINT`
- `TINYINT`

此外，内部类型 `U128`（`OID`）在 v0.1 中没有 SQL 关键字映射。

尝试使用任何不支持的类型创建列将导致 "not yet implemented" 错误。
