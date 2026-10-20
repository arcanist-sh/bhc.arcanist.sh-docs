+++
title = "bhc-target"
description = "Target specifications"
weight = 50
+++

# bhc-target

Target platform specifications for the Basel Haskell Compiler.

## Overview

This crate describes target machines: architecture, vendor, OS, ABI, endianness, CPU features and the LLVM data layout string. It parses target triples and detects the host. Dependencies: `serde`, `serde_json`, `thiserror`, `rustc-hash`.

## Key Types

```rust
pub struct TargetSpec {
    pub arch: Arch,
    pub vendor: Vendor,
    pub os: Os,
    pub abi: Abi,
    pub endianness: Endianness,
    pub features: CpuFeatures,
    pub data_layout: String,
}

pub struct CpuFeatures {
    pub sse42: bool,
    pub avx: bool,
    pub avx2: bool,
    pub avx512: bool,
    pub neon: bool,
    pub sve: bool,
    pub simd128: bool,
}
```

| Enum | Variants |
|------|----------|
| `Arch` | `X86_64`, `Aarch64`, `Wasm32`, `Wasm64`, `Riscv32`, `Riscv64`, `Nvptx64`, `Amdgcn` |
| `Vendor` | `Unknown`, `Apple`, `Pc` |
| `Os` | `Linux`, `MacOs`, `Windows`, `FreeBsd`, `Wasi`, `None` |
| `Abi` | `Gnu`, `Musl`, `Msvc`, `None` |

`CpuFeatures::default_for_arch` turns on SSE4.2 for x86_64, NEON for aarch64 and SIMD128 for wasm. All targets are little-endian.

## Parsing Triples

`parse_triple` expects at least three dash-separated parts, `<arch>-<vendor>-<os>[-<abi>]`. Accepted spellings:

| Part | Accepted values |
|------|-----------------|
| arch | `x86_64`/`amd64`, `aarch64`/`arm64`, `wasm32`, `wasm64`, `riscv32`, `riscv64`, `nvptx64`/`nvptx`, `amdgcn`/`gcn` |
| vendor | `unknown`, `apple`, `pc` |
| os | `linux`, `darwin`/`macos`, `windows`/`win32`, `freebsd`, `wasi`, `none` |
| abi | `gnu`, `musl`, `msvc` (optional) |

`host_target()` builds the spec for the machine the compiler runs on. The `targets` module has constructors for `x86_64-unknown-linux-gnu`, `x86_64-apple-darwin`, `aarch64-unknown-linux-gnu`, `aarch64-apple-darwin` and `wasm32-unknown-wasi`.

## How the Driver Uses It

`bhc-driver` routes on the `--target` string itself before consulting this crate:

- A triple containing `wasm` (for example `wasm32-wasi`) selects the WebAssembly backend.
- A triple containing `cuda`, `nvptx`, `ptx` or `amdgcn` selects the GPU path.
- Otherwise the LLVM native backend is used, with the `TargetSpec` from `parse_triple`.

If `parse_triple` rejects the string, the driver falls back to `host_target()`. Two-part triples such as `wasm32-wasi`, and triples with an unlisted vendor such as `nvptx64-nvidia-cuda`, do not parse, but still reach the right backend through the substring checks.

## Usage

```bash
# Native (host)
bhc Main.hs

# WebAssembly (WASI)
bhc --target wasm32-wasi Main.hs
```
