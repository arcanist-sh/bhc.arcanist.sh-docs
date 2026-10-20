+++
title = "WebAssembly"
description = "Compiling to WebAssembly with WASI"
weight = 2
+++

bhc compiles Haskell to WebAssembly modules that use the WASI interface (`wasi_snapshot_preview1`). On bhc's differential suite, 236 of 243 programs produce output byte-identical to the native backend.

## Target

```bash
bhc --target wasm32-wasi Main.hs -o main.wasm
```

`--target wasm` is accepted as a shorthand. Other spellings, including `wasi` on its own, do not select the WebAssembly backend and produce a native executable instead.

## Profiles

The [edge profile](@/docs/profiles/edge.md) gives the module a smaller linear memory: 256 KB initial and 4 MB maximum, against 1 MB and 16 MB by default.

```bash
bhc --profile=edge --target wasm32-wasi Main.hs -o main.wasm
```

## Running WASM Modules

### With wasmtime

```bash
wasmtime main.wasm
```

Any runtime that implements `wasi_snapshot_preview1` should work; wasmtime is the one bhc is tested against.

### File access

`readFile` reads host files through WASI. A path is resolved against the first directory preopened by the runtime, so grant one with `--dir`:

```bash
wasmtime --dir . main.wasm
```

Without a preopen, the program cannot see host files. `writeFile` and `appendFile` are always visible to later reads in the same program; writing their contents through to the host filesystem is still being completed.

### In the browser

The module imports WASI functions, so a browser host needs a WASI shim (any library that implements `wasi_snapshot_preview1`) passed in as the `wasi_snapshot_preview1` import object. bhc does not ship such a shim.

The [online playground](https://arcanist.sh/bhc/playground/) is a separate build: it runs bhc's interpreter compiled to WebAssembly, not a module produced by `--target wasm32-wasi`.

## Exports

A compiled module exports `_start`, its linear `memory`, and a set of runtime helper functions. `foreign export` declarations are not supported for the WebAssembly target yet: a module without `main` fails to compile.

## Limitations

- Single-threaded only
- File access only through a `--dir` preopen
- The `Handle` API (`openFile`, `hGetLine`, `hClose`, ...) and `System.Directory` are not available yet
- `getArgs` does not see the arguments passed by the runtime yet
- No network access
- Linear memory is capped at 16 MB (4 MB with the edge profile; 32 MB with the numeric profile)

## Binary Size

WebAssembly output is small: a hello-world module is under 4 KB. Standard tools such as Binaryen's `wasm-opt` work on the output:

```bash
wasm-opt -O3 main.wasm -o main-opt.wasm
```
