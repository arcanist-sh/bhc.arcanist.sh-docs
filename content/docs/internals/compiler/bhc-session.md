+++
title = "bhc-session"
description = "Compilation session"
weight = 60
+++

# bhc-session

Compilation session management for the Basel Haskell Compiler.

## Overview

This crate holds the state for one compilation: the `Options` set from the command line, search paths, the set of loaded modules, and the working directory. `bhc-driver` creates a session and reads it in every phase; `bhc-core`, `bhc-codegen`, `bhc-wasm`, `bhc-gpu`, `bhc-linker`, `bhc-package`, `bhc-lsp` and `bhc-playground` also use its `Profile` and `OutputType` types.

## Key Types

| Type | Description |
|------|-------------|
| `Session` | `options`, `search_paths`, loaded-module set, working directory |
| `SessionRef` | `Arc<Session>`, created by `create_session` |
| `Options` | All compiler options (see below) |
| `SearchPaths` | `module_paths`, `interface_paths` (`.bhi`), `library_paths` |
| `Profile` | Compilation profile |
| `OptLevel` | `None`, `Less`, `Default`, `Aggressive`, `Size`, `SizeMin` |
| `OutputType` | `Object`, `StaticLib`, `DynamicLib`, `Executable`, `LlvmIr`, `LlvmBitcode`, `Assembly`, `Wasm` |
| `DebugInfo` | `None`, `LineTablesOnly`, `Full` |
| `IrDumpOptions` | Flags for dumping AST, Core, Tensor IR, Loop IR and LLVM IR |
| `SessionError` | `ConfigNotFound`, `InvalidConfig`, `UnsupportedTarget`, `Io` |

## Session Configuration

```rust
let session = Session::new(Options {
    profile: Profile::Numeric,
    opt_level: OptLevel::Default,
    target_triple: Some("wasm32-wasi".to_string()),
    output_type: OutputType::Executable,
    ..Options::default()
})?;
```

Besides the fields above, `Options` carries import and library paths, `warn_all`/`deny_warnings`, `emit_kernel_report`, `tensor_fusion`, `stdlib_path`, `hackage_packages` (`--package`), separate-compilation settings (`compile_only`, `output_object_dir`, `output_interface_dir`, `package_dbs`, `package_ids`), enabled `extensions`, and `cpp_defines` for the CPP preprocessor.

## Profiles

| Profile | Description | Helpers |
|---------|-------------|---------|
| `Default` | Standard lazy evaluation | |
| `Server` | Server workloads | |
| `Numeric` | Strict by default; runs the Tensor IR fusion pipeline | `is_strict_by_default`, `has_fusion_guarantees` |
| `Edge` | Minimal runtime footprint | `is_strict_by_default` |
| `Realtime` | Bounded pauses, per-frame arenas (not exposed on the CLI) | |
| `Embedded` | No GC; escaping allocations are rejected (not exposed on the CLI) | `is_strict_by_default`, `requires_escape_analysis`, `is_gc_free` |

The CLI accepts `--profile default|server|numeric|edge`. See [Profiles](@/docs/profiles/_index.md) for what each one does today.
