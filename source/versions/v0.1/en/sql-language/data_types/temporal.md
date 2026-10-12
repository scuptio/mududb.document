# Temporal Types

```sql
DATE                    -- calendar date (year, month, day)
TIME(p)                 -- time of day, optional fractional-second precision
TIMESTAMP(p)            -- date + time, no time zone
TIMESTAMPTZ(p)          -- date + time, with time zone
```

The precision parameter `p` specifies the number of fractional-second digits (0–6). When omitted, no precision metadata is stored.

SQL text in v0.1 has no dedicated temporal literal syntax (neither typed literals like `DATE '2024-01-01'` nor `::date` casts are accepted).
