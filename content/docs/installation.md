+++
title = "Installation"
description = "Install bhc, the Basel Haskell Compiler, with the install script, from a release archive, or through hx."
weight = 1
+++

bhc is one compiler binary plus the runtime and standard libraries it links into your programs. Releases are built for two platforms:

| Platform | Release archive |
|---|---|
| macOS on Apple silicon | `bhc-aarch64-darwin.tar.gz` |
| Linux on x86_64 | `bhc-x86_64-linux.tar.gz` |

Native compilation also needs a system C toolchain for linking (the Xcode Command Line Tools on macOS, `cc` from your distribution on Linux).

## Install script

```bash
curl -fsSL https://arcanist.sh/bhc/install.sh | sh
```

The script downloads the latest release for your platform and installs it to `~/.bhc`: the compiler in `~/.bhc/bin` and the runtime libraries in `~/.bhc/lib`. Set `BHC_INSTALL_DIR` to install somewhere else.

It then prints the two lines to add to your shell profile, so that your shell finds `bhc` and the linker finds the runtime:

```bash
export PATH="$HOME/.bhc/bin:$PATH"
export LIBRARY_PATH="$HOME/.bhc/lib:$LIBRARY_PATH"
```

## From a release archive

Every [GitHub release](https://github.com/arcanist-sh/bhc/releases) carries the archives and a `SHA256SUMS.txt`; compare the output of `shasum -a 256 bhc-aarch64-darwin.tar.gz` (or `sha256sum` on Linux) with the line for your archive. Unpack the archive for your platform anywhere you like, then put its `bin` directory on `PATH` and its `lib` directory on `LIBRARY_PATH`, as above.

## With hx

[hx](https://arcanist.sh/hx/) installs and manages bhc toolchains next to GHC:

```bash
hx toolchain install --bhc 0.2.22    # or --bhc latest
```

Then build a project with bhc, once or for good:

```bash
hx build --backend bhc
```

```toml
# hx.toml
[compiler]
backend = "bhc"
```

See [Using bhc with hx](@/docs/quickstart.md#build-a-project-with-hx) for the bhc settings hx understands.

## Check the installation

```bash
bhc --version
```

prints the installed version, for example `bhc 0.2.22`.

## Troubleshooting

**`bhc: command not found`.** The `bin` directory is not on your `PATH`. Add the `export PATH=…` line above to `~/.zshrc` or `~/.bashrc` and open a new shell.

**`library 'bhc_rts' not found`, or other link errors.** The linker cannot find the runtime libraries. Add the `export LIBRARY_PATH=…` line above.

**Unsupported platform.** Intel Macs, Linux on arm64 and Windows have no release archive yet. On those machines you can build bhc from [source](https://github.com/arcanist-sh/bhc) with a Rust toolchain and LLVM.

## Uninstall

```bash
curl -fsSL https://arcanist.sh/bhc/install.sh | sh -s -- --uninstall
```

asks for confirmation, removes `~/.bhc`, and offers to remove the lines the installer added to your shell profile.

## Next steps

- Compile your first program with the [Quick Start](@/docs/quickstart.md)
- Choose a [runtime profile](@/docs/profiles/_index.md)
- Build for [WebAssembly](@/docs/targets/wasm.md)
- Browse the [API reference](https://docs.arcanist.sh/bhc/api/) for the Prelude and standard library
