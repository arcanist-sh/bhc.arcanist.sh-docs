+++
title = "Edge Profile"
description = "Minimal runtime for WASM and embedded targets"
weight = 4
+++

The **edge** profile is meant for small, sandboxed deployments. Today it changes the configuration of WebAssembly output: smaller memory limits and no debug names.

## When to Use

Use the edge profile when:

- Targeting WebAssembly (WASI)
- You want the module's memory capped at a few megabytes
- Building short-lived, single-threaded programs

## Usage

```bash
bhc --profile=edge --target=wasm32-wasi Main.hs -o main.wasm
wasmtime main.wasm
```

## Example

```haskell
module Main where

fibonacci :: Int -> Int
fibonacci n
    | n <= 1    = n
    | otherwise = fibonacci (n - 1) + fibonacci (n - 2)

main :: IO ()
main = print (fibonacci 25)
```

```bash
bhc --profile=edge --target=wasm32-wasi Fib.hs -o fib.wasm
wasmtime fib.wasm
# 75025
```

The module for this program is about 4 KB. The WebAssembly backend emits WASI modules directly, so small programs are small under every profile; the edge profile mainly lowers the memory ceiling.

## What it changes

For `--target=wasm32-wasi` (`crates/bhc-wasm/src/lib.rs`, `WasmConfig::edge_profile`):

| Setting | Edge | Default |
|---------|------|---------|
| Initial memory | 256 KB | 1 MB |
| Maximum memory | 4 MB | 16 MB |
| Debug names | off | on |
| SIMD128 | on | only if the compiler was built with the `simd128` feature |

For native builds the edge profile skips demand analysis and worker/wrapper, and is otherwise the same as the default profile.

## Runtime Characteristics

- **Threading**: Single-threaded
- **Memory**: Linear memory capped at 4 MB; compiled programs do not collect garbage yet, so this suits short-lived runs
- **Host interface**: WASI preview 1 (`wasi_snapshot_preview1`), checked with wasmtime

## Current limits

- A module must have a `main`. Exporting individual functions to the host with `foreign export` is not supported yet; the compiler reports `no main binding found`.
- Modules are tested under wasmtime. Other WASI hosts (serverless platforms, edge compute services) should work if they support WASI preview 1, but they are not part of the test suite.

See [WebAssembly](@/docs/targets/wasm.md) for what the WebAssembly backend covers.
