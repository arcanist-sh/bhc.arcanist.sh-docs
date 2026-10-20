+++
title = "bhc-tensor-ir"
description = "Tensor IR for numeric optimization"
weight = 32
+++

# bhc-tensor-ir

Tensor Intermediate Representation for the Basel Haskell Compiler.

## Overview

Tensor IR represents array and tensor operations together with their shape, strides, layout and aliasing, so that a fusion pass can merge chains of operations into kernels. It sits between Core IR and [Loop IR](@/docs/internals/compiler/bhc-loop-ir.md). Dependencies include `bhc-core` (the input) and `bhc-types`.

## When Used

The driver runs this pipeline only with `--profile numeric` or `--tensor-fusion`:

1. `lower::lower_module(&CoreModule) -> Vec<TensorOp>` recognizes tensor operations in Core bindings.
2. `fusion::fuse_ops(&mut FusionContext, ops) -> Vec<Kernel>` groups them into kernels; the driver always creates the context in strict mode (`FusionContext::new(true)`).
3. `fusion::generate_kernel_report` produces the data behind `--kernel-report`.

The resulting kernels go to Loop IR lowering (and from there into WASM functions on the WASM target) and to the GPU path for CUDA/ROCm targets. The native LLVM backend compiles from Core IR and does not consume these kernels, so on native the pipeline currently affects only the kernel report.

## Tracked Properties

Every `TensorRef` carries a `TensorMeta`:

| Field | Type | Description |
|-------|------|-------------|
| `dtype` | `DType` | `Bool`, `Int8`..`Int64`, `UInt8`..`UInt64`, `Float16`, `Float32`, `Float64`, `BFloat16`, `Complex64`, `Complex128` |
| `shape` | `Shape` | Dimension sizes (`Dim`, static or dynamic) |
| `strides` | `Strides` | Byte strides per dimension |
| `layout` | `Layout` | `Contiguous`, `Strided`, `Tiled(TileInfo)` |
| `alias` | `Option<BufferId>` | Buffer this tensor references |

## Operations

```rust
pub enum TensorOp {
    Constant(ConstantOp),
    Unary(UnaryOp, TensorRef),
    Binary(BinaryOp, TensorRef, TensorRef),
    Map(MapFn, TensorRef),
    ZipWith(ZipFn, TensorRef, TensorRef),
    Reduce(ReduceOp, Axis, TensorRef),
    ReduceAll(ReduceOp, TensorRef),
    Scan(ReduceOp, Axis, TensorRef),
    Fold(FoldFn, TensorRef, TensorRef),
    Reshape(Shape, TensorRef),
    Slice(SliceSpec, TensorRef),
    Transpose(Permutation, TensorRef),
    Broadcast(Shape, TensorRef),
    Concat(Axis, Vec<TensorRef>),
    Split(Axis, Vec<usize>, TensorRef),
    MatMul(TensorRef, TensorRef),
    BatchMatMul(TensorRef, TensorRef),
    Dot(TensorRef, TensorRef),
    Outer(TensorRef, TensorRef),
    Conv(ConvSpec, TensorRef, TensorRef),
    Gather(Axis, TensorRef, TensorRef),
    Scatter(Axis, TensorRef, TensorRef, TensorRef),
}
```

A `Kernel` has inputs, outputs, a `KernelBody` (`Fused(Vec<TensorOp>)` or `LoopNest`), allocations and `FusionInfo` recording the fusion decisions.

## Fusion Patterns

`FusionPattern` covers the four patterns the H26 spec requires to fuse:

```haskell
map f (map g xs)                -- MapMap
zipWith f (map g a) (map h b)   -- ZipWithMaps
sum (map f xs)                  -- ReduceMap
foldl' op z (map f xs)          -- FoldMap
```

It also recognizes ML-style compositions: `Softmax`, `LogSoftmax`, `LayerNorm`, `RMSNorm`, `Attention`, `Gelu`, `Silu` and `FusedLinear`. These patterns fuse in the crate's tests; because native code generation does not use the fused kernels, the spec's fusion guarantee is not yet delivered for native executables.
