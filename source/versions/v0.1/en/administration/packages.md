# Application Packages

MuduDB applications are deployed as `.mpk` packages. An `.mpk` file is a ZIP archive that contains the compiled WebAssembly components, schema DDL, procedure descriptors, optional initialization data, and application metadata.

## Package contents

A valid `.mpk` archive contains at least:

| Entry | Required | Description |
|-------|----------|-------------|
| `package.cfg.json` | Yes | Application metadata (`name`, `lang`, `version`, `use_async`). |
| `package.desc.json` | Yes | Module/procedure descriptor (`ModProcDesc`). |
| `ddl.sql` | Yes | Schema DDL statements. |
| `initdb.sql` | Packager: yes; loader: no | Initial data SQL. The loader accepts a missing or empty file. |
| `package.manifest.json` | No | Forward/backward-compatibility manifest. New packages should include it. |
| `*.wasm` | Packager: yes; loader: no | One or more compiled WebAssembly component modules. |

See the [MPK Manifest Contract](../reference/contracts/mpk_manifest_v1.md) for the full specification.

## Building a package

Use `mpm-build` from the application directory:

```bash
mpm-build create \
  --package-cfg package.cfg.json \
  --package-desc package.desc.json \
  --ddl-sql ddl.sql \
  --initdb-sql initdb.sql \
  --wasm-files wallet.wasm \
  --output wallet.mpk
```

The output is a `.mpk` file that can be installed into a running `mudud` server.

## Installing a package

Use `mcli` over the HTTP management endpoint:

```bash
mcli --http-addr 127.0.0.1:8300 app-install --mpk target/wasm32-wasip2/release/wallet.mpk
```

## Listing installed applications

```bash
mcli --http-addr 127.0.0.1:8300 app-list
```

## Viewing application details

List procedures in an app:

```bash
mcli --http-addr 127.0.0.1:8300 app-detail --app wallet
```

Show one procedure:

```bash
mcli --http-addr 127.0.0.1:8300 app-detail --app wallet --module wallet --proc transfer_funds
```

`--proc` must be used together with `--module`.

## Uninstalling a package

```bash
mcli --http-addr 127.0.0.1:8300 app-uninstall --app wallet
```

## Invoking a procedure

```bash
mcli --addr 127.0.0.1:9527 --http-addr 127.0.0.1:8300 app-invoke \
  --app wallet \
  --module wallet \
  --proc transfer_funds \
  --json '{"from_user_id": 1001, "to_user_id": 1002, "amount": 1200}'
```

See [Your First MPK Package](../tutorial/tutorials/wallet.md) for a complete example, and [mcli Management Interface (HTTP)](../reference/mcli-admin.md) for all management commands.

