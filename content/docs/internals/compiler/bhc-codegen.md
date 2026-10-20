+++
title = "bhc-codegen"
description = "Native code generation"
weight = 51
+++

# bhc-codegen

Native code generation for the Basel Haskell Compiler.

## Overview

This crate lowers Core IR to LLVM IR through [inkwell](https://github.com/TheDan64/inkwell) (0.8, LLVM 21) and emits object files, assembly, LLVM IR or bitcode. Nearly all of the work lives in `llvm/lower.rs`, which turns Core expressions, closures, constructors, `case` and dictionary-passed class methods into LLVM functions that call into the RTS (`bhc_alloc` and other `bhc_*` symbols).

`llvm/loop_lower.rs` can lower Loop IR to LLVM (`lower_loop_ir`, `lower_loop_irs`), but the native driver does not use it today: numeric-profile programs are compiled from Core like any other program.

## Pipeline

```
Core IR
   │
   ▼
┌──────────────┐
│  Lowering    │  llvm::lower_core_module*  (Core → LLVM IR)
└──────────────┘
   │
   ▼
┌──────────────┐
│  LLVM        │  LlvmModule::verify / optimize (new pass manager)
└──────────────┘
   │
   ▼
Object file (LlvmModule::write_to_file)
```

## Key Components

| Item | Description |
|------|-------------|
| `CodegenBackend`, `CodegenContext`, `CodegenModule` | Backend-neutral traits |
| `CodegenConfig` | Target, opt level, debug info, PIC, LTO, CPU (`for_target`, `with_*` builders) |
| `CodegenOutputType` | `Object`, `Assembly`, `LlvmIr`, `LlvmBitcode` |
| `llvm::LlvmBackend`, `LlvmContext`, `LlvmModule` | inkwell-backed implementation |
| `llvm::Lowering`, `ConstructorMeta`, `CompiledSymbol` | Core lowering state, constructor layouts, exported symbols for other modules |
| `llvm::TypeMapper` | Maps BHC types to LLVM types |
| `set_io_primitives`, `is_io_primitive` | Registry of IO primitives used by the IO closure mode |

## Usage

```rust
use bhc_codegen::{CodegenConfig, CodegenOutputType, LlvmContext, LlvmModule};
use bhc_codegen::llvm::lower_core_module;

let ctx = LlvmContext::new(CodegenConfig::for_target(target))?;
let module = LlvmModule::new(&ctx, "Main")?;

lower_core_module(&ctx, &module, &core_module)?;
module.verify()?;
module.optimize(&ctx, opt_level)?;
module.write_to_file(Path::new("Main.o"), CodegenOutputType::Object)?;
```

For separate compilation the driver calls `lower_core_module_multimodule_with_constructors`, which mangles symbols with the module name, declares externs for imported symbols and registers constructor metadata from imported modules. Newtype constructors flagged `is_newtype` are compiled as the identity.

## Runtime Interaction

- Heap objects are allocated with `bhc_alloc` from the RTS. Compiled code does not use the garbage collector: there are no safepoints or stack maps, and memory allocated by compiled code is not reclaimed.
- Calls in tail position are marked as LLVM tail calls.
- Uncaught exceptions and `exitWith` go through RTS entry points.

## Optimization Levels

`LlvmModule::optimize` maps `bhc_session::OptLevel` to LLVM pipelines:

| `OptLevel` | LLVM pipeline |
|------------|---------------|
| `None` | no passes |
| `Less`, `Size`, `SizeMin` | `default<O1>` |
| `Default` | `default<O2>` |
| `Aggressive` | `default<O3>` |
