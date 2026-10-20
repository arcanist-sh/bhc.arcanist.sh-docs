+++
title = "Compilation Targets"
description = "bhc compilation targets for different platforms"
sort_by = "weight"
+++

bhc compiles Haskell to two kinds of output today:

- **Native** - executables for the host machine, through LLVM. Release binaries of bhc are published for aarch64 macOS (Apple silicon) and x86_64 Linux.
- **WASM** - WebAssembly modules that run under a WASI runtime such as wasmtime, with `--target wasm32-wasi`.

GPU code generation (CUDA, ROCm) is validated against a mock device only, and the Apple Metal runtime is a library inside the compiler rather than a compilation target. See the [targets overview](https://arcanist.sh/bhc/targets/) for the full picture.
