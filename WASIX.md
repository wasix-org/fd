# fd for WASIX

Standalone `fd` from upstream **10.5.0** (base `4f81778774463bf414a184cbe6d5219ad2229646`).
No Pi code, JavaScript runtime, or Bash wrapper is included.

## Build and test

Install Rustup, Python 3.9+, `patch`, a native C compiler, **Wasmer 7.4.2**,
and **wasm-tools 1.251.0**. On macOS the C compiler comes with Xcode command
line tools. Install the pinned cargo-wasix and WASIX compiler once:

```sh
cargo install cargo-wasix --version 0.1.33 --locked
cargo wasix download-toolchain v2026-07-07.3+rust-1.96
```

On Linux, restore the executable bits missing from the pinned archive's native
linker shims (needed to compile Cargo build scripts):

```sh
chmod +x "$(rustc +wasix --print sysroot)"/lib/rustlib/*-unknown-linux-gnu/bin/gcc-ld/*
```

Then clone the fork and build:

```sh
git clone --branch codex/wasix https://github.com/wasix-org/fd.git
cd fd
bash wasix/build.sh
python3 wasix/test.py
```

`build.sh` runs `python3 wasix/prepare.py`, then
**`cargo wasix build --release --locked --bin fd`**. Cargo-wasix handles
the compiler, Binaryen 130 and exception conversion. The script disables wide
arithmetic for browser compatibility, removes debug metadata, validates the
result, collects license notices, and runs `wasmer package build`.

You can also run `python3 wasix/prepare.py` followed by `cargo wasix build
--release --locked` directly. Use `build.sh` for the browser-compatible release
flags and WebC packaging. Dependencies are fixed by the root `Cargo.lock` and
`.cargo/config.toml`; the tiny portability patches are applied to
checksum-verified crate archives under ignored `.wasix/deps/`.

Outputs: `.wasix/dist/fd.wasm` and `.wasix/fd-10.5.1.webc`.
`.wasix/provenance.json` records the source revision, dirty state, compiler,
Cargo/cargo-wasix versions, Wasmer version, and lockfile hash. Artifact hashes
and byte sizes are printed by the build.

The [WASIX workflow](.github/workflows/wasix.yml) builds and tests on Linux,
rebuilds into a fresh Cargo target directory, compares both artifacts byte for
byte, and uploads them with SHA-256 checksums. Reproduce that check with:

```sh
cp .wasix/dist/fd.wasm .wasix/first.wasm
cp .wasix/fd-10.5.1.webc .wasix/first.webc
CARGO_TARGET_DIR=.wasix/repro-target bash wasix/build.sh
cmp .wasix/first.wasm .wasix/dist/fd.wasm
cmp .wasix/first.webc .wasix/fd-10.5.1.webc
```

The byte comparison uses the same checkout and host toolchain. Cross-host
compiler distributions are not assumed to produce identical output. Check out
a release tag and use its recorded versions to reproduce a published release.

## Port patches

Fd adds WASI file-type/byte-string extensions and enables `--one-file-system`.
It uses the default WASIX SIGINT handler because `ctrlc` requires Unix APIs.
Shell completions are included. WASI metadata cannot distinguish named pipes,
and Unix-only ownership filters are unavailable.

`ignore` supplies directory traversal and filesystem IDs; `same-file` uses
WASI device/inode metadata for identity and symlink-loop detection; `walkdir`
uses WASI filesystem IDs for its single-threaded walker. Only the necessary
WASI paths are added; Unix/Windows behavior is preserved. Versions and archive
checksums are in `wasix/dependencies.json`, patches in `wasix/patches/`.
No Wasmer, libc, or libuv changes are required.

Tests execute the actual WebC and cover searches, ignore rules, Unicode,
spaces, symlinks, filesystem limits, and exit behavior. Commands launched by
`fd --exec` must be supplied by the surrounding environment.

## Use as a package

```sh
wasmer run wasmer/fd@10.5.1 --volume "$PWD:/workspace" -- --no-require-git --glob "*.rs" /workspace
wasmer run --registry wasmer.wtf wasmer/fd@10.5.1 -- --version
```

Other Wasmer packages can reference the command without embedding its binary:

```toml
[dependencies]
"wasmer/fd" = "=10.5.1"

[[command]]
name = "fd"
module = "wasmer/fd:fd"
runner = "wasi"
```

## Publish

Build and test from a clean, committed checkout. Authenticate with an account
that can publish in the `wasmer` namespace, then upload the same staged package:

```sh
wasmer login --registry wasmer.io
wasmer publish . --registry wasmer.io --wait=container --non-interactive
wasmer login --registry wasmer.wtf
wasmer publish . --registry wasmer.wtf --wait=container --non-interactive
```

Packages: [wasmer.io](https://wasmer.io/wasmer/fd) and
[wasmer.wtf](https://wasmer.wtf/wasmer/fd). To update, change `wasmer.toml`
and `wasix/build.json` together and publish a new version. Keep license notices
in the package. CI does not publish or require registry credentials.

Package 10.5.1 stores dependency notices at `/opt/fd/licenses`. This keeps
license mounts out of `/usr`, where they interfere with Wasmer command installation.
The compiled utility versions are unchanged.
