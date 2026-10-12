# 安装

## 使用 mudup 安装

`mudup` 是计划中的安装器，目前尚不可用；下列命令描述的是计划中的发布版安装路径。现阶段可用的安装方式是从源码构建。

```bash
curl --proto '=https' --tlsv1.2 -fsSL https://github.com/scuptio/mudup/releases/download/latest/mudup-init.sh | sh
mudup install
```

验证已安装的工具：

```bash
mudud --version
mcli --version
mpm-build --version
mpm-crate --version
mpm-install --version
mgen --version
mtp --version
```

## 从源码构建

开发构建建议使用 Ubuntu 或 Debian。

```bash
sudo apt-get update -y
sudo apt-get install -y python3 python3-pip python-is-python3 build-essential curl liburing-dev clang libclang-dev llvm-dev pkgconf
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup toolchain install 1.99.0
rustup default 1.99.0
rustup target add wasm32-wasip2
python -m pip install toml tomli-w
cargo install cargo-make
```

从源码仓库安装工作区二进制文件：

```bash
python script/build/install_binaries.py
```

## 配置文件

`mudud` 按以下顺序加载找到的第一个配置文件：`--cfg` / `-c` 指定的路径（如提供）、当前工作目录下的 `./mudud.cfg`、用户主目录下的 `~/.mududb/mudud.cfg`。如果都不存在，`mudud` 将无法启动。

`mudud` 不会自动创建配置文件。在不启动服务器的情况下生成默认的 `./mudud.cfg`：

```bash
mudud init-cfg
```
