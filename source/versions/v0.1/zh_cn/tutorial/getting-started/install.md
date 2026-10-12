# 安装

从 MuduDB 源码工作区构建。`mudup` 在仓库中仍是设计提案，不能作为已可用的安装路径。

## 从源码构建

以下命令面向 Ubuntu/Debian。先通过 rustup 安装 Rust，再从 MuduDB 源码根目录执行仓库命令。根目录本身不是 Cargo workspace，`cargo make` 负责协调各独立工作区。

```bash
sudo apt-get update
sudo apt-get install -y git python3 python3-pip python3-venv build-essential curl liburing-dev clang libclang-dev llvm-dev pkg-config
# Run these in the MuduDB source checkout:
rustup show
rustup target add wasm32-wasip2
cargo install cargo-make --locked
python3 -m venv .venv/build
.venv/build/bin/python -m pip install toml tomli-w
.venv/build/bin/python script/build/install_binaries.py --profile release
```

`rust-toolchain.toml` 当前固定 Rust `1.99.0`，`rustup show` 会选择该工作区固定的工具链。安装脚本将 `mudud`、`mcli`、`mgen`、`mtp`、`mpm`、`mpm-build`、`mpm-crate`、`mpm-install` 安装到 Cargo 的二进制目录，请确保该目录位于 `PATH`。

如需完整构建仓库及 Rust Wallet 包，激活 Python 环境后执行 `cargo make`。其他 guest 语言需要额外工具链，见 [SDK 与执行模式](../../programming/sdk.md)。

```bash
mudud --version
mcli --version
mpm --help
mpm-build --version
mpm-crate --version
mpm-install --version
mgen --version
mtp --version
```

## 初始配置

在新的运行目录中执行：

```bash
mkdir -p mudud-run
cd mudud-run
mudud init-cfg
mudud serve --cfg ./mudud.cfg
```

`init-cfg` 会覆盖 `./mudud.cfg`，应在修改配置前运行。生成配置默认使用 `Tokio`，数据和应用包路径分别为 `./data`、`./mpk`。显式指定 `--cfg` / `-c` 时只读取该路径，文件缺失会报错。未指定时依次查找 `./mudud.cfg`、`~/.mududb/mudud.cfg`。服务端不会自动生成缺失的配置文件。
