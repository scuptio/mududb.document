# Wallet Example

`example/mpk/wallet` shows how to package business procedures into an `.mpk`, install the package into MuduDB, and call procedures through `mcli app-invoke`.

## Build the Package

Run from the repository root:

```bash
(cd example/mpk/wallet && cargo make package)
```

Generated package:

```text
target/wasm32-wasip2/release/wallet.mpk
```

## Install the Application

```bash
mcli --http-addr 127.0.0.1:8300 app-install --mpk target/wasm32-wasip2/release/wallet.mpk
mcli --http-addr 127.0.0.1:8300 app-list
```

## Invoke a Procedure

```bash
mcli --addr 127.0.0.1:9527 --http-addr 127.0.0.1:8300 app-invoke --app wallet --module wallet --proc create_user --json '{
  "user_id": 1001,
  "name": "Alice",
  "email": "alice@example.com"
}'
```

Procedure calls are sent through TCP. `--http-addr` is still used to fetch procedure metadata.
