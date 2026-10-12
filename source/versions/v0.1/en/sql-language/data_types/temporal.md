# Temporal Types

```sql
DATE                    -- calendar date (year, month, day)
TIME(p)                 -- time of day, optional fractional-second precision
TIMESTAMP(p)            -- date + time, no time zone
TIMESTAMPTZ(p)          -- date + time, with time zone
```

The precision parameter `p` specifies the number of fractional-second digits (0–6). When omitted, no precision metadata is stored.

SQL text in v0.1 has no dedicated temporal literal syntax (neither typed literals like `DATE '2024-01-01'` nor `::date` casts are accepted).

Canonical quoted strings are converted to the target temporal column type during binding, including in primary-key predicates. Invalid text returns `TypeConversionFailed`; declared fractional precision is applied by the type codec. For example:

```sql
CREATE TABLE calendar (d DATE PRIMARY KEY, note TEXT);
INSERT INTO calendar VALUES ('2026-05-20', 'release');
SELECT note FROM calendar WHERE d = '2026-05-20';
```
