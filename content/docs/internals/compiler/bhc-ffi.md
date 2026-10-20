+++
title = "bhc-ffi"
description = "FFI support"
weight = 65
+++

# bhc-ffi

Rust-side Foreign Function Interface primitives for the Basel Haskell Compiler.

## Overview

This crate is a runtime library, not a compiler pass. It provides pinned buffers that are safe to hand to C code, a pluggable BLAS abstraction, and small matrix helpers built on them. It depends on `bhc-rts-alloc`, `bhc-rts-gc`, `bhc-tensor-ir` and `libc`; `bhc-gpu` uses its `FfiSafe` trait for device transfers.

Haskell-level `foreign import` declarations are not handled here: they are parsed into `ast::ForeignDecl`, carried through HIR as `ForeignDecl` and into `CoreModule::foreign_imports`, and `bhc-codegen` declares the C symbol and marshals arguments from the Haskell type (unwrapping `IO`).

## Modules

| Module | Contents |
|--------|----------|
| `pinned` | `PinnedBuffer<T>`, `PinnedSlice`, `PinnedSliceMut`; `with_pinned`, `with_pinned_mut`, `with_pinned_buffer`, `with_pinned_buffer_mut` |
| `blas` | `BlasProvider` trait, `FallbackBlas` (pure Rust), `openblas::OpenBlas` (feature `openblas`), `default_provider`, `should_use_blas`, `BLAS_THRESHOLD`, `Transpose`, `Layout` |
| `tensor` | `Matrix<T>` over a `PinnedBuffer`, `matmul`/`smatmul`, `dot`/`sdot` |
| crate root | `FfiError`, `FfiResult`, `FfiSafe` marker trait, `is_aligned`, `is_aligned_to` |

## Usage

```rust
use bhc_ffi::{blas, with_pinned, PinnedBuffer};

// Copy into a pinned buffer and run the closure with its pointer
// (with_pinned_mut copies the result back into the slice)
let sum = with_pinned(&data, |ptr, len| unsafe { c_sum(ptr, len) })?;

// Allocate a pinned buffer directly
let buf: PinnedBuffer<f64> = PinnedBuffer::from_slice(&[1.0, 2.0, 3.0])?;

// BLAS through the configured provider
let provider = blas::default_provider();   // OpenBlas with `openblas`, else FallbackBlas
let d = provider.ddot(&buf, &buf)?;
```

`FfiSafe` is an `unsafe` marker trait (`Copy + 'static`, with a `C_TYPE_NAME`) implemented for the primitive integer and float types (`i8`..`i64`, `u8`..`u64`, `isize`, `usize`, `f32`, `f64`, ...); only `FfiSafe` element types can live in a `PinnedBuffer`.

## Foreign Imports in Haskell

```haskell
foreign import ccall "sin"
  c_sin :: Double -> Double

foreign import ccall unsafe "fast_op"
  fastOp :: CInt -> CInt
```

`safe`/`unsafe` annotations are parsed (`ForeignSafety`). `foreign export` declarations are also parsed, but HIR-to-Core lowering does not yet populate `CoreModule::exports`, so no C-callable wrapper is emitted.

## BLAS

`FallbackBlas` implements the provider in Rust. The `openblas` Cargo feature links OpenBLAS; an `accelerate` feature is declared in `Cargo.toml` but has no implementation yet. `should_use_blas(m, n, k)` returns true when any dimension reaches `BLAS_THRESHOLD` (64), so small multiplications avoid FFI overhead.
