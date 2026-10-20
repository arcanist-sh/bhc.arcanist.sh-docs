+++
title = "bhc-loop-ir"
description = "Loop IR for vectorization"
weight = 33
+++

# bhc-loop-ir

Loop Intermediate Representation for the Basel Haskell Compiler.

## Overview

Loop IR makes iteration structure explicit: loops with bounds and steps, SSA values, memory references with access patterns, and per-loop metadata for vectorization and parallelization. It is produced from fused Tensor IR kernels when compiling with the Numeric profile or `--tensor-fusion`.

## Pipeline Position

```
Tensor IR kernels  →  [lower_kernels]  →  Loop IR  →  [VectorizePass / ParallelPass]  →  kernel report
                                                    →  [bhc_wasm::lower::lower_loop_ir]  →  WASM functions
```

The WASM backend lowers Loop IR kernels into WASM functions. `bhc-codegen` contains an LLVM lowering (`llvm::loop_lower`), but the native driver path does not call it: native executables are built from Core IR, so Loop IR currently has no effect on native output.

## Modules

| Module | Contents |
|--------|----------|
| `lib.rs` | IR types (`LoopIR`, `Loop`, `Stmt`, `Value`, `Op`, `MemRef`, `LoopType`, `TargetArch`) |
| `lower` | `lower_kernel`, `lower_kernels`, `LowerConfig`, `LowerError` (Tensor IR `Kernel` → `LoopIR`) |
| `vectorize` | `VectorizePass`, `VectorizeConfig`, `VectorizeReport`, `SimdIntrinsic` |
| `parallel` | `ParallelPass`, `ParallelConfig`, `ParallelStrategy` (`Static`, `Dynamic`, `Guided`), `ParFor`, `ParMap`, `ParReduce`, `Range` |

## Key Types

```rust
pub struct LoopIR {
    pub name: Symbol,
    pub params: Vec<Param>,
    pub return_ty: LoopType,
    pub body: Body,
    pub allocs: Vec<Alloc>,
    pub loop_info: Vec<LoopMetadata>,
}

pub struct Loop {
    pub id: LoopId,
    pub var: ValueId,
    pub lower: Value,   // inclusive
    pub upper: Value,   // exclusive
    pub step: Value,
    pub body: Body,
    pub attrs: LoopAttrs,
}

pub enum Stmt {
    Assign(ValueId, Op),
    Loop(Loop),
    If(IfStmt),
    Store(MemRef, Value),
    Call(Option<ValueId>, Symbol, Vec<Value>),
    Return(Option<Value>),
    Barrier(BarrierKind),
    Comment(String),
}
```

`LoopAttrs` is a bitflag set (`PARALLEL`, `VECTORIZE`, `UNROLL`, `REDUCTION`, `INDEPENDENT`, `TILED`, `TILE_INNER`). `LoopMetadata` records trip count (`Static`, `Dynamic`, `Bounded`), vector width, parallel chunk size, unroll factor and inter-loop dependencies. SIMD types include `LoopType::VEC4F32`, `VEC8F32`, `VEC2F64` and `VEC4F64`.

## Passes

| Pass | What it does |
|------|--------------|
| `VectorizePass` | Decides per loop whether it is vectorizable for the `TargetArch` (SSE, SSE2, AVX, AVX2, NEON, Generic) and at what width |
| `ParallelPass` | Decides which loops can run in parallel and with which strategy; the driver uses deterministic settings |

In the driver both passes run as analyses whose results go into the kernel report (`--kernel-report`). Tiling and unrolling exist only as attributes and metadata fields; there is no pass that performs them.
