# Wallet 示例

`example/mpk/wallet` 展示了如何将业务过程打包为 `.mpk`，将包安装到 MuduDB 中，并通过 `mcli app-invoke` 调用过程。

## 构建包

在仓库根目录执行：

```bash
(cd example/mpk/wallet && cargo make package)
```

生成的包：

```text
target/wasm32-wasip2/release/wallet.mpk
```

## 安装应用程序

```bash
mcli --http-addr 127.0.0.1:8300 app-install --mpk target/wasm32-wasip2/release/wallet.mpk
mcli --http-addr 127.0.0.1:8300 app-list
```

## 调用过程

```bash
mcli --addr 127.0.0.1:9527 --http-addr 127.0.0.1:8300 app-invoke --app wallet --module wallet --proc create_user --json '{
  "user_id": 1001,
  "name": "Alice",
  "email": "alice@example.com"
}'
```

过程调用通过 TCP 发送。`--http-addr` 仍用于获取过程元数据。
