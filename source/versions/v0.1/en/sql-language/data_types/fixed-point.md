# Fixed-Point Types

```sql
NUMERIC(precision, scale)
DECIMAL(precision, scale)
NUMERIC        -- without parameters
DECIMAL        -- without parameters
```

`NUMERIC` stores fixed-point decimal values with a maximum precision of 38 digits; without parameters it defaults to `NUMERIC(38, 0)`. The `(precision, scale)` parameters are enforced on write: values are rounded (round-half-even) to the declared scale, and values whose digit count exceeds the declared precision are rejected.

Examples:

```sql
amount NUMERIC(18, 2)
rate   DECIMAL(9, 4)
```
