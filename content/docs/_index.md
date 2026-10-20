+++
title = "Documentation"
description = "The documentation for bhc, the Basel Haskell Compiler."
sort_by = "weight"
+++

**bhc**, the Basel Haskell Compiler, is a clean-slate compiler for the Haskell you already write. It is written in Rust, compiles to native code through LLVM and to WebAssembly for WASI, and lets you choose the runtime profile a program is built for.

## What is bhc?

- **Compatibility first**: Haskell 2010 plus the common GHC extensions, with type classes, transformer stacks and Template Haskell; every module of Pandoc type-checks and compiles
- **Runtime profiles**: one language, with a `default`, `server`, `numeric` or `edge` runtime chosen per build with `--profile` or in `hx.toml`
- **Several targets**: native binaries through LLVM, WASI modules that run in `wasmtime`, and a Metal runtime for Apple silicon GPUs
- **Transparency**: every intermediate representation can be printed, and the numeric profile reports its kernels
- **hx integration**: build Cabal projects with bhc by setting `backend = "bhc"` in [hx](https://arcanist.sh/hx/)

New here? Start with [Installation](@/docs/installation.md) and the [Quick Start](@/docs/quickstart.md). The overview, the measured [status](https://arcanist.sh/bhc/status/) and the blog live on [arcanist.sh/bhc](https://arcanist.sh/bhc/). Type signatures and function documentation for the Prelude and standard library are in the [API reference](https://docs.arcanist.sh/bhc/api/).
