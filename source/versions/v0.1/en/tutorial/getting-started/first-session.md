# First Session

## Start the Server

`mudud` requires a configuration file and does not create one automatically; generate a default `./mudud.cfg` first if none exists:

```bash
mudud init-cfg
ulimit -n 65535
mudud
```

When running under `systemd`, configure `LimitNOFILE=65535` at the service level.

## Open the Interactive SQL Shell

```bash
mcli --addr 127.0.0.1:9527 shell --app demo
```

Run a complete CRUD flow:

```sql
CREATE TABLE users_demo (
  id INT PRIMARY KEY,
  name TEXT
);

INSERT INTO users_demo (id, name) VALUES (1, 'Alice');
SELECT id, name FROM users_demo WHERE id = 1;

UPDATE users_demo SET name = 'Alice-Updated' WHERE id = 1;
DELETE FROM users_demo WHERE id = 1;
```

Exit the shell:

```text
\q
```

## Next Step

Continue with {doc}`../tutorials/wallet` to build, install, and invoke a procedure application.
