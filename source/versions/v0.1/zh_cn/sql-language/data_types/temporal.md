# 时间类型

```sql
DATE                    -- calendar date (year, month, day)
TIME(p)                 -- time of day, optional fractional-second precision
TIMESTAMP(p)            -- date + time, no time zone
TIMESTAMPTZ(p)          -- date + time, with time zone
```

精度参数 `p` 指定小数秒位数（0–6）。省略时，不存储精度元数据。

v0.1 的 SQL 文本没有专门的时间字面量语法（`DATE '2024-01-01'` 形式的类型化字面量和 `::date` 转换均不可用）。

绑定时会将规范格式的字符串字面量转换为目标时间列类型，主键谓词也使用相同转换。无效文本返回 `TypeConversionFailed`，声明的小数秒精度由类型编解码器应用。例如：

```sql
CREATE TABLE calendar (d DATE PRIMARY KEY, note TEXT);
INSERT INTO calendar VALUES ('2026-05-20', 'release');
SELECT note FROM calendar WHERE d = '2026-05-20';
```
