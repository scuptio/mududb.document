# MuduDB Server Configuration Contract v1

## Scope

This document specifies the `mudud.cfg` server configuration file format. The file controls server listen ports, execution mode, runtime paths, and io_uring behavior.

## Version history

| Version | Date | Summary |
|---------|------|---------|
| 1 | 2026-6-25 | Initial TOML configuration. No explicit `version` field; compatibility is implicit through serde defaults and aliases. |

## File location

An explicit `--cfg` / `-c` path is used exclusively: a missing or invalid file fails without fallback. Without that option, the server tries `./mudud.cfg`, then `~/.mududb/mudud.cfg`; if neither exists it returns `NotFound`. Relative storage paths are resolved from the server process working directory.

## Fields and validation

See [Configuration Reference](../../administration/configuration.md) for all root fields and the `[invocation_limits]`, `[replication]`, and `[backup]` tables, including defaults and validation rules.

## Compatibility

There is no serialized `version` field. Only fields annotated with Serde defaults can be omitted; required fields still fail decoding. Unknown root fields are currently ignored, whereas `[invocation_limits]` rejects unknown fields. An arbitrary `version = 2` does not select a new parser or guarantee rejection, so operators must check compatibility with the target binary before rollback.

`server_mode` accepts case-insensitive `Legacy`, `IOUring` (also `io_uring`), and `Tokio`; routing accepts `ConnectionId` / `connection_id`, `PlayerId` / `player_id`, and `RemoteHash` / `remote_hash`. `component_target` accepts `p2` or `p3`.

`db_path` and `page_size` are persistent settings. Pointing at another directory selects another database; changing the page size of existing data requires migration or reinitialization. Most settings require restart. The mutability classification of `tcp_multi_port` does not itself provide a configuration reload command.

## Implementation

- `crates/db-kernel/mudu_runtime/src/backend/mudud_cfg/{mod,io,replication}.rs`
- `crates/db-kernel/runtime_wasm/src/service/invocation_limits.rs`
