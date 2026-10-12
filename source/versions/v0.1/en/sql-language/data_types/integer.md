# Integer Types

```sql
INT      -- 32-bit signed
BIGINT   -- 64-bit signed
HUGEINT  -- 128-bit signed
```

`INT` maps to the internal 32-bit signed integer (`I32`). `BIGINT` maps to the 64-bit signed integer (`I64`). `HUGEINT` maps to the internal 128-bit signed integer (`I128`).

All integer literals in SQL are parsed as `I64` by default; literals outside the `I64` range are rejected by the parser. The parser performs no per-column range checking: an oversized value written to a narrower integer column is converted by the runtime's truncating cast rules (for example, `3000000000` written to an `INT` column wraps to `-1294967296`).
