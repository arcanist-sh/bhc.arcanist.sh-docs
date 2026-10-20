+++
title = "Internals"
description = "BHC compiler internals and crate documentation"
weight = 10
sort_by = "weight"
+++

# Compiler Internals

This section documents the internal architecture of the Basel Haskell Compiler. BHC is implemented in Rust as a Cargo workspace: compiler crates under `crates/`, the runtime system under `rts/`, Rust-side standard library support under `stdlib/` (with Haskell sources in each crate's `hs/` directory and the `H26.*` modules in `stdlib/H26/`), and command-line tools under `tools/`.

## Architecture Overview

```
Source Code
    │
    ▼
┌─────────────┐
│   Parser    │  bhc-lexer, bhc-parser, bhc-ast
└─────────────┘
    │
    ▼
┌─────────────┐
│    HIR      │  bhc-lower, bhc-hir   (name resolution, desugaring, .bhi imports)
└─────────────┘
    │
    ▼
┌─────────────┐
│ Type Check  │  bhc-types, bhc-typeck
└─────────────┘
    │
    ▼
┌─────────────┐
│   Core IR   │  bhc-hir-to-core, bhc-core
│             │  dictionaries, pattern compilation, simplifier,
│             │  worker/wrapper, specialization, monomorphization
└─────────────┘
    │
    ├──────────────────────────────┐
    │                              │ (Numeric profile)
    │                              ▼
    │                       ┌─────────────┐
    │                       │  Tensor IR  │  bhc-tensor-ir
    │                       └─────────────┘
    │                              │
    │                              ▼
    │                       ┌─────────────┐
    │                       │   Loop IR   │  bhc-loop-ir
    │                       └─────────────┘
    │                              │  kernels: reports, WASM, GPU
    ▼                              ▼
┌──────────────────────────────────────────┐
│ Backend   native: bhc-codegen (LLVM)     │
│           wasm:   bhc-wasm               │
│           gpu:    bhc-gpu (PTX, MSL, ...) │
└──────────────────────────────────────────┘
    │
    ▼
┌─────────────┐
│   Linker    │  bhc-linker (native)
└─────────────┘
    │
    ▼
Executable / .wasm / .ptx
```

`bhc-driver` runs this pipeline and `bhc-session` holds its options. Native and WASM code are generated from Core; on the native path the Tensor IR and Loop IR kernels feed the kernel report but are not compiled into the executable. With `-c`, each module also produces a `.bhi` interface (`bhc-interface`) and a `.bhc` sidecar of Core bodies used for cross-module specialization.

## Crate Categories

### [Compiler Crates](@/docs/internals/compiler/_index.md)

The main compiler implementation, including parsing, type checking, IR transformations, and code generation.

### [Runtime Crates](@/docs/internals/runtime/_index.md)

The BHC Runtime System (RTS) under `rts/`: `bhc-rts`, `bhc-rts-alloc`, `bhc-rts-arena`, `bhc-rts-gc` and `bhc-rts-scheduler`. Compiled native code allocates through `bhc_alloc` and does not currently use the garbage collector.

### [Standard Library](@/docs/internals/stdlib/_index.md)

Rust-side support for the BHC standard library (`bhc-prelude`, `bhc-base`, `bhc-containers`, `bhc-text`, `bhc-transformers`, `bhc-numeric`, `bhc-concurrent`, `bhc-system`, `bhc-utils`), together with the Haskell sources they back.

### [Tools](@/docs/internals/tools/_index.md)

The `bhc` compiler binary (`crates/bhc`), the `bhci` REPL and `bhi` IR inspector (`tools/`), the browser playground (`crates/bhc-playground`), and the `bhc-docs` documentation generator (`tools/bhc-docs`). `tools/pandoc-harness` holds the shell and Python scripts used to compile Pandoc with BHC.

## Key Design Principles

- **Modularity**: Each crate has a focused responsibility with clean interfaces
- **Type Safety**: Typed indices and IR types make invalid states harder to build
- **Transparency**: Each IR can be dumped and inspected for debugging and optimization
- **Separate compilation**: Modules compile independently against interface files
