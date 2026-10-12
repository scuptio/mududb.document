# Data Types

MuduDB stores values as typed objects. The SQL parser maps SQL type names to internal scalar kinds. Each type has a fixed or variable in-memory representation and well-defined comparison and serialization behaviour.

## Overview

Types are divided into scalar and complex categories:

- **Scalar** — single atomic values (numbers, strings, temporal values).
- **Complex** — `ARRAY`, `RECORD`, and `BINARY`.

All scalar types support ordering and equality comparison. Complex types have structural comparison rules.

The following table summarises the SQL type names recognised by the parser and their internal mappings.

| SQL Type | Internal Kind | Notes |
|---|---|---|
| `INT` | `I32` | 32-bit signed integer |
| `BIGINT` | `I64` | 64-bit signed integer |
| `HUGEINT` | `I128` | 128-bit signed integer |
| `FLOAT` | `F32` | 32-bit IEEE-754 float |
| `DOUBLE`, `REAL` | `F64` | 64-bit IEEE-754 float |
| `NUMERIC(p,s)` | `Numeric` | Fixed-point decimal (precision up to 38 digits) |
| `DECIMAL(p,s)` | `Numeric` | Alias for `NUMERIC` |
| `CHAR(n)` | `String` | String with declared length (no padding, not enforced) |
| `VARCHAR(n)` | `String` | Variable-length string |
| `TEXT` | `String` | Unbounded text |
| `DATE` | `Date` | Calendar date |
| `TIME(p)` | `Time` | Time of day with optional fractional-second precision |
| `TIMESTAMP(p)` | `Timestamp` | Date+time without time zone |
| `TIMESTAMPTZ(p)` | `TimestampTz` | Date+time with time zone |

```{toctree}
:maxdepth: 1

integer
floating-point
fixed-point
string
temporal
other
```

## Nullability

Every column is nullable by default unless declared `NOT NULL` or marked as part of a `PRIMARY KEY`. `NULL` is a valid literal in `INSERT` and `UPDATE` statements.

```sql
CREATE TABLE example (
    id   INT PRIMARY KEY,        -- implicitly NOT NULL
    name CHAR(32) NOT NULL,      -- explicitly NOT NULL
    note VARCHAR(200)            -- nullable by default
);
```

## Unsupported Types

The following types are recognised by the grammar but not yet implemented in the SQL parser:

- `BOOLEAN`
- `SMALLINT`
- `TINYINT`

Additionally, the internal type `U128` (`OID`) has no SQL keyword mapping in v0.1.

Attempting to create a column with any unsupported type results in a "not yet implemented" error.
