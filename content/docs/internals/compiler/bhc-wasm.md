+++
title = "bhc-wasm"
description = "WebAssembly backend"
weight = 53
+++

# bhc-wasm

WebAssembly backend for the Basel Haskell Compiler.

## Overview

This crate compiles Core IR to WebAssembly without LLVM. It builds the module in memory and writes the binary itself (`WasmModule::write_wasm`, `to_wasm`; `to_wat` for text). The output is a WASI module with a `_start` entry point that runs in a WASI runtime such as wasmtime. In the differential test suite, 236 of 243 fixtures produce byte-identical output on the WASM and native backends.

## Pipeline

`bhc-driver` selects this backend when the `--target` string contains `wasm`:

1. `WasmModule::new(name, WasmConfig::for_profile(profile), target)`
2. `add_wasi_imports` and `add_runtime_functions` (allocator, printing, string and list helpers, file IO, argument and environment access)
3. `core_lower::lower_core_module(&core, &mut module, &runtime_indices)` lowers the program
4. With the Numeric profile or `--tensor-fusion`, `lower::lower_loop_ir` adds each Loop IR kernel as a function (SIMD128 when enabled)
5. `add_start_function` wires `_start` to call `main` and then `proc_exit`
6. `write_wasm` writes `<Module>.wasm`, or the `-o` path

## Modules

| Module | Responsibility |
|--------|----------------|
| `core_lower` | Core IR to WASM instructions: closures, constructors, pattern matching, dictionaries, strings, IO |
| `lower` | Loop IR kernels to WASM functions |
| `wasi` | WASI imports and generated runtime functions (`generate_*`) |
| `codegen` | `WasmModule`, `WasmCodegenContext`, `RuntimeIndices`, binary emitter, SIMD128 mapping, value types |
| `runtime` | `RuntimeConfig`, `LinearMemory`/`MemoryLayout`, `WasmArena`, and a mark-sweep `gc` module (not wired into generated code) |

`WasmBackend` also implements `bhc_codegen::CodegenBackend` for `wasm32`/`wasm64` target specs.

## Usage

```bash
# Compile to WASM
bhc --target wasm32-wasi Main.hs -o app.wasm

# Run with wasmtime; --dir grants access to host files
wasmtime --dir . app.wasm
```

## WASI Imports

All imports come from `wasi_snapshot_preview1`: `fd_write`, `fd_read`, `proc_exit`, `args_sizes_get`, `args_get`, `environ_sizes_get`, `environ_get`, `path_open` and `fd_close`.

Host files are opened relative to the first `--dir` preopen. `writeFile` writes to the host file (when a preopen exists) and to an in-memory file table; `readFile` checks the table first, then the host file, and raises if neither has the path. See [WebAssembly](@/docs/targets/wasm.md) for what the target supports.

## Configuration

| `WasmConfig` constructor | Used for | Memory (initial / max) |
|--------------------------|----------|------------------------|
| `default()` | Default and Server profiles | 16 / 256 pages (1 MB / 16 MB) |
| `edge_profile()` | Edge profile; no debug names, size-optimized | 4 / 64 pages |
| `for_profile(Numeric)` | Numeric profile | 32 / 512 pages |
| `browser_profile()` | Not selected by the driver | 16 / 1024 pages |

Cargo features: `simd128` (default) enables SIMD128 in the default config; `browser` is declared but gates no code yet.

## Runtime

Memory is allocated with a bump allocator over linear memory (the `heap_ptr` global). Generated code does not collect garbage, so memory use grows with allocation until the maximum page count is reached. There is no threading support and no JavaScript glue; the browser [playground](@/docs/internals/tools/bhc-playground.md) runs a separate interpreter build of the front end rather than modules produced by this crate.
