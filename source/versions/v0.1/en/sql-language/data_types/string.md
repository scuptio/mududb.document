# String Types

```sql
CHAR(n)      -- string with declared length n (not padded, not enforced)
VARCHAR(n)   -- string with declared length n (not enforced)
TEXT         -- unbounded text
```

All three map to the internal `String` type. MuduDB uses UTF-8 encoding. The length parameter `n` is stored in column metadata but does not truncate inserted values.

String literals in SQL are single-quoted: `'hello'`.
