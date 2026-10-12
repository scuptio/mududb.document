# 应用包

MuduDB 应用以 `.mpk` 包的形式部署。`.mpk` 是一个 ZIP 归档，包含编译后的 WebAssembly 组件、schema DDL、过程描述符、可选初始化数据及应用元数据。

## 包内容

有效的 `.mpk` 归档至少包含以下条目：

| 条目 | 必需 | 说明 |
|------|------|------|
| `package.cfg.json` | 是 | 应用元数据（`name`、`lang`、`version`、`use_async`）。 |
| `package.desc.json` | 是 | 模块/过程描述符（`ModProcDesc`）。 |
| `ddl.sql` | 是 | Schema DDL 语句。 |
| `initdb.sql` | 打包器：是；加载器：否 | 初始数据 SQL。加载器接受缺失或为空的文件。 |
| `package.manifest.json` | 否 | 前后向兼容清单。新包应包含。 |
| `*.wasm` | 打包器：是；加载器：否 | 一个或多个编译后的 WebAssembly 组件模块。 |

完整规范请参见 [MPK 清单契约](../reference/contracts/mpk_manifest_v1.md)。

## 构建包

在应用目录下使用 `mpm-build`：

```bash
mpm-build create \
  --package-cfg package.cfg.json \
  --package-desc package.desc.json \
  --ddl-sql ddl.sql \
  --initdb-sql initdb.sql \
  --wasm-files wallet.wasm \
  --output wallet.mpk
```

输出为一个 `.mpk` 文件，可安装到运行中的 `mudud` 服务器。

## 安装包

通过 HTTP 管理端点使用 `mcli`：

```bash
mcli --http-addr 127.0.0.1:8300 app-install --mpk target/wasm32-wasip2/release/wallet.mpk
```

## 列出已安装应用

```bash
mcli --http-addr 127.0.0.1:8300 app-list
```

## 查看应用详情

列出应用中的过程：

```bash
mcli --http-addr 127.0.0.1:8300 app-detail --app wallet
```

查看单个过程：

```bash
mcli --http-addr 127.0.0.1:8300 app-detail --app wallet --module wallet --proc transfer_funds
```

`--proc` 必须与 `--module` 一起使用。

## 卸载包

```bash
mcli --http-addr 127.0.0.1:8300 app-uninstall --app wallet
```

## 调用过程

```bash
mcli --addr 127.0.0.1:9527 --http-addr 127.0.0.1:8300 app-invoke \
  --app wallet \
  --module wallet \
  --proc transfer_funds \
  --json '{"from_user_id": 1001, "to_user_id": 1002, "amount": 1200}'
```

完整示例请参见[你的第一个 MPK 包](../tutorial/tutorials/wallet.md)，所有管理命令请参见 [mcli HTTP 管理接口](../reference/mcli-admin.md)。
