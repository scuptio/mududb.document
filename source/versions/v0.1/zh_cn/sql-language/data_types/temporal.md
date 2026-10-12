# 时间类型

```sql
DATE                    -- calendar date (year, month, day)
TIME(p)                 -- time of day, optional fractional-second precision
TIMESTAMP(p)            -- date + time, no time zone
TIMESTAMPTZ(p)          -- date + time, with time zone
```

精度参数 `p` 指定小数秒位数（0–6）。省略时，不存储精度元数据。

v0.1 的 SQL 文本没有专门的时间字面量语法（`DATE '2024-01-01'` 形式的类型化字面量和 `::date` 转换均不可用）。
