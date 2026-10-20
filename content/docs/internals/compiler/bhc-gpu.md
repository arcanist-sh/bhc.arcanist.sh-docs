+++
title = "bhc-gpu"
description = "GPU code generation"
weight = 52
+++

# bhc-gpu

GPU code generation and runtime support for the Basel Haskell Compiler.

## Overview

This crate generates GPU kernel source from Tensor IR kernels and provides device, memory, transfer and launch abstractions. Kernel generators exist for five targets; one runtime, Apple Metal, executes on real hardware.

## Supported Backends

| Backend | Codegen module | Output | Runtime | Status |
|---------|----------------|--------|---------|--------|
| NVIDIA CUDA | `codegen::ptx` | PTX | `runtime::cuda` (feature `cuda`, driver API loaded with `libloading`) | Emitted; validated against a mock device only |
| AMD ROCm | `codegen::amdgcn` | AMDGCN | `runtime::rocm` (feature `rocm`) | Emitted; mock only |
| Apple Metal | `codegen::metal` | MSL | `runtime::metal` (feature `metal`, macOS) | Runs on Apple silicon |
| Vulkan/OpenCL | `codegen::spirv` | SPIR-V | none | Codegen only |
| WebGPU | `codegen::wgsl` | WGSL | none | Codegen only |

`DeviceKind` has a variant for each of these plus `Mock`. When no runtime finds a device, `available_devices()` returns a mock device so the backend stays usable in tests and on CPU-only machines.

## Pipeline

```
Tensor IR kernel
    │
    ▼
┌──────────────┐
│ GpuLowering  │  lower: suitability analysis (GpuSuitability), launch shape
└──────────────┘
    │
    ▼
┌──────────────┐
│  codegen::*  │  PTX / AMDGCN / MSL / SPIR-V / WGSL source
└──────────────┘
    │
    ▼
┌──────────────┐
│  runtime::*  │  load, launch (CUDA, ROCm, Metal)
└──────────────┘
```

In the driver, a GPU target or PTX emission request (`is_gpu_target` / `is_ptx_emit`) writes a `.ptx` file built from the module's Tensor IR kernels with `codegen::ptx`. Metal is not yet wired into the driver.

## Key Types

| Type | Description |
|------|-------------|
| `DeviceId`, `DeviceInfo`, `DeviceKind` | Device enumeration (`available_devices`, `select_device`, `default_device`) |
| `GpuContext` | Allocation, copies (sync and async), streams, `compile_kernel`, `launch`, synchronization |
| `DeviceBuffer<T>`, `DevicePtr` | Device memory for `bhc_ffi::FfiSafe` element types |
| `Transfer`, `TransferHandle`, `TransferQueue` | Asynchronous host/device transfers |
| `GpuKernel`, `KernelConfig`, `LaunchConfig` | Compiled kernels and grid/block configuration |
| `GpuBackend` | `bhc_codegen::CodegenBackend` implementation for GPU targets |
| `runtime::metal::MetalRuntime` | Compiles MSL at runtime and runs element-wise `f32` kernels on shared-memory buffers |

## Memory Management

```rust
use bhc_gpu::{select_device, DeviceBuffer, DeviceId};

let ctx = select_device(DeviceId(0))?;
let mut d_buf: DeviceBuffer<f32> = ctx.alloc(1024)?;

ctx.copy_to_device(&host_data, &mut d_buf)?;
let kernel = ctx.compile_kernel(&tensor_ir_kernel)?;
ctx.launch(&kernel, launch_config, &[d_buf.as_ptr()])?; // CUDA/ROCm when enabled; else a logged mock launch
ctx.copy_to_host(&d_buf, &mut host_out)?;
```

`GpuContext::launch` dispatches to the CUDA or ROCm runtime when the matching feature is on; Metal execution currently goes through `MetalRuntime::run_elementwise_f32` directly. Reductions and matrix multiplication on Metal are not implemented yet.
