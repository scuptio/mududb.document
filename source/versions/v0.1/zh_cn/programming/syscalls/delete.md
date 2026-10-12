# delete

从会话级键值存储中删除一个键。

## 可用性

KV `delete` 系统调用已在 guest/host 的 WIT 边界中声明并由内核宿主实现，但 `sys_interface` 尚未提供面向过程内 Rust 代码的 `mudu_delete` 封装。独立（standalone）SDK 门面以如下形式暴露它：

```rust
pub async fn delete(oid: &UniOid, key: &[u8]) -> Result<UniResponse<()>, ApiError>;
```

在封装落地之前，过程代码可改用 `command`（`DELETE` 语句）删除关系行。

## 概念签名

`uni-syscall.wit` 中的线格式签名：

```text
delete: func(oid: uni-oid, key: list<u8>) -> result<_, uni-error>
```

## 参数

- `oid` —— 要修改其键值存储的会话。
- `key` —— 原始键字节序列。

## 行为

- 从会话的键值存储中删除该键。与 `put` 一样，当前会话事务打开时删除在事务中暂存，否则直接生效。

## 返回值

- 成功时返回 `()`。

## 错误

- `EntityNotFound`：`oid` 不对应一个存活会话。

## 事务语义

- 参与当前会话事务；事务外由宿主直接应用删除（自动提交）。

## 注意

- `delete` 是 `read` / `write` / `range-scan` 的 KV 对应操作，与通过 `command` 执行的 SQL `DELETE` 语句无关。
