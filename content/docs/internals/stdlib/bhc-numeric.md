+++
title = "bhc-numeric"
description = "Numeric computing"
weight = 4
+++

# bhc-numeric

Numeric computing primitives for the Basel Haskell Compiler.

## Overview

This crate is a Rust library of numeric building blocks: SIMD vector types, dense vectors, matrices and tensors, a BLAS abstraction, matrix decompositions, sparse matrices and a tensor arena, with about 80 C functions for compiled code to call. The intended Haskell API is in `hs/BHC/Numeric/` (`Vector`, `Matrix`, `Tensor`, `Tensor.Shaped`, `SIMD`, `BLAS`, `Sparse`, `Einsum`, `Grad`).

**Status:** the Rust code is implemented and covered by its own tests (`tests/{vector,matrix,tensor,blas,simd,fusion}_tests.rs`), but it is not yet connected to compiled Haskell. `libbhc_numeric.a` is not on the native link line, generated code calls none of its functions, and the `BHC.Numeric.*` modules are not compiled into programs. The numeric profile's fusion work happens in the compiler (`bhc-tensor-ir`, `bhc-loop-ir`) and does not call this crate. `Data.Vector` in a Haskell program is a separate set of compiler builtins, not this crate's `Vector`.

## Rust modules

| Module | Contents |
|--------|----------|
| `simd` | `Vec2F32`, `Vec4F32`, `Vec8F32`, `Vec2F64`, `Vec4F64`, `Vec4I32`, `Vec8I32`; x86-64 SSE, AVX, AVX2 and FMA intrinsics chosen at run time with `is_x86_feature_detected!`, with scalar fallbacks; `SimdFeatures` |
| `simd_neon` | `NeonVec4F32`, `NeonVec2F64`, `NeonVec4I32` (aarch64 only) |
| `vector` | `Vector<T>`: element-wise add/sub/mul, scale, dot, norm, normalize, sum, mean |
| `matrix` | `Matrix<T>`: zeros, identity, get, add, scale, transpose, matmul, trace, norm |
| `tensor` | `Tensor<T>` with `Shape` and `Layout`; sum, mean, norm |
| `blas` | `PureRustBlas`, `SimdRustBlas`, `BlasBackend` selection; `ddot`, `daxpy`, `dscal`, `dnrm2`, `dgemm` |
| `blas_openblas` | OpenBLAS provider (feature `openblas`) |
| `blas_accelerate` | Apple Accelerate provider (feature `accelerate`, macOS) |
| `decomp` | LU (with `determinant`, `solve`), QR (`solve`, `pseudoinverse`), Cholesky, SVD (Golub-Reinsch, with `rank`, `condition_number`, `pseudoinverse`, `low_rank_approx`) |
| `sparse` | `CsrMatrix<T>`: sparse matrix-vector (`spmv`) and matrix-matrix (`spmm`) products |
| `arena` | `TensorArena`, `ArenaTensor`: bump-allocated tensor temporaries |

The default feature set is `simd`. A `mkl` feature is declared and `BlasBackend` names MKL, but there is no MKL provider module yet. GPU support is not in this crate; see the compiler's `bhc-gpu` crate.

## C interface

| Prefix | Functions (`_f32` / `_f64` variants) |
|--------|--------------------------------------|
| `bhc_simd_` | `dot`, `sum`, `saxpy`, `features` |
| `bhc_vector_` | `new`, `from`, `get`, `len`, `data`, `add`, `sub`, `mul`, `scale`, `dot`, `norm`, `normalize`, `sum`, `mean`, `free` |
| `bhc_matrix_` | `from`, `zeros`, `identity`, `rows`, `cols`, `get`, `data`, `add`, `scale`, `transpose`, `matmul`, `trace`, `norm`, `free` |
| `bhc_tensor_` | `from`, `sum`, `mean`, `norm`, `free` |
| `bhc_blas_` | `ddot`, `daxpy`, `dscal`, `dnrm2`, `dgemm`, `provider_name` |
| `bhc_sparse_csr_` | `new`, `spmv`, `spmm`, `free` |
| `bhc_arena_` | `new`, `alloc`, `zeros`, `used`, `available`, `full`, `reset`, `clone`, `free` |
