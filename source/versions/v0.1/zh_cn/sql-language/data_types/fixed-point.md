# 定点类型

```sql
NUMERIC(precision, scale)
DECIMAL(precision, scale)
NUMERIC        -- without parameters
DECIMAL        -- without parameters
```

`NUMERIC` 存储定点十进制值，最大精度为 38 位；省略参数时默认为 `NUMERIC(38, 0)`。`(precision, scale)` 参数在写入时强制执行：值按 `scale` 四舍五入（round-half-even）到声明的小数位数，舍入后总位数超出 `precision` 的值会被拒绝。

示例：

```sql
amount NUMERIC(18, 2)
rate   DECIMAL(9, 4)
```
