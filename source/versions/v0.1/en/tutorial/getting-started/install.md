# Installation

Build from the MuduDB source checkout. `mudup` is a design proposal, not an available installation path in this repository.

## Source build

The following commands target Ubuntu/Debian. Install Rust with rustup first, then run the repository commands from the MuduDB checkout root. The root is not a Cargo workspace; `cargo make` orchestrates the independent workspaces.

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

`rust-toolchain.toml` currently pins Rust `1.99.0`; `rustup show` selects the checkout's pinned toolchain. The install script installs `mudud`, `mcli`, `mgen`, `mtp`, `mpm`, `mpm-build`, `mpm-crate`, and `mpm-install` into Cargo's binary directory. Ensure that directory is on `PATH`.

For the full repository build, including the Rust Wallet package, activate the Python environment and run `cargo make`. Additional guest languages have their own toolchains; see [SDKs and Execution Modes](../../programming/sdk.md).

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

## Initial configuration

In a new run directory:

```bash
mkdir -p mudud-run
cd mudud-run
mudud init-cfg
mudud serve --cfg ./mudud.cfg
```

`init-cfg` overwrites `./mudud.cfg`, so run it before editing your configuration. It selects `Tokio` by default, with `./data` and `./mpk` paths. An explicit `--cfg` / `-c` path is exclusive: a missing file is an error. Without that option the server searches `./mudud.cfg`, then `~/.mududb/mudud.cfg`. The server does not generate a missing configuration automatically.
