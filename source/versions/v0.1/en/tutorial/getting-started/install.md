# Installation

## Install with mudup

`mudup` is a planned installer and is not available yet; the commands below describe the planned release-install path. The working installation path today is building from source.

```bash
curl --proto '=https' --tlsv1.2 -fsSL https://github.com/scuptio/mudup/releases/download/latest/mudup-init.sh | sh
mudup install
```

Verify the installed tools:

```bash
mudud --version
mcli --version
mpm-build --version
mpm-crate --version
mpm-install --version
mgen --version
mtp --version
```

## Build from Source

Ubuntu or Debian is recommended for development builds.

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

Install workspace binaries from the source repository:

```bash
python script/build/install_binaries.py
```

## Configuration File

`mudud` loads the first configuration file it finds, in this order: the `--cfg` / `-c` path (if supplied), `./mudud.cfg` in the current working directory, then `~/.mududb/mudud.cfg` in the user's home directory. If none of them exist, `mudud` fails to start.

`mudud` does not create a configuration file automatically. To generate a default `./mudud.cfg` without starting the server:

```bash
mudud init-cfg
```
