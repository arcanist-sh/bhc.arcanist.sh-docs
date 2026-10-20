+++
title = "bhc-linker"
description = "Linking support"
weight = 54
+++

# bhc-linker

Linker support for the Basel Haskell Compiler.

## Overview

This crate runs the final link step for the native backend. It does not implement a linker itself: it builds a command line for the platform's linker driver and invokes it as a subprocess. Dependencies are `bhc-target` (for `TargetSpec`/`Os`) and `bhc-session` (for `OutputType`).

## Key Types

| Item | Purpose |
|------|---------|
| `LinkerConfig` | Target, output type and path, objects, libraries, search paths, `pie`/`lto`/`strip`, extra flags |
| `LinkLibrary` | A library by name (`named`, `static_lib`) or explicit path (`with_path`) |
| `LinkOutputType` | `Executable`, `StaticLib`, `DynamicLib` (converted from `bhc_session::OutputType`) |
| `Linker` trait | `name`, `supports_target`, `executable`, `build_args`, `link` |
| `SystemLinker` | Default implementation, chosen per target OS |
| `LldLinker` / `LldFlavor` | LLD in `Gnu`, `Darwin`, `Msvc` or `Wasm` flavor |
| `LinkerError` | `LinkerNotFound`, `LinkerFailed`, `ExecutionFailed`, `InvalidConfig` |

```rust
let config = LinkerConfig::new(target, "a.out")
    .output_type(LinkOutputType::Executable)
    .with_objects(objects)
    .with_library(LinkLibrary::named("bhc_rts"));
bhc_linker::link(&config)?;
```

`link` calls `linker_for_target`, which currently always returns a `SystemLinker`; `LldLinker` exists but is not selected automatically.

## Platform Linkers

| Platform | `SystemLinker` executable | Argument style |
|----------|---------------------------|----------------|
| Linux | `cc` | Unix (`-o`, `-L`, `-l`, `-Wl,-Bstatic`, `-pie`, `-flto`, `-s`) |
| macOS | `cc` (ld64 underneath) | Unix |
| Windows | `link.exe` | MSVC (`/OUT:`, `/LIBPATH:`, `/DLL`) |
| WASI | `wasm-ld` | `--no-entry --export-all` |

The WebAssembly backend (`bhc-wasm`) writes complete modules itself and does not go through this crate.

## Libraries Linked by the Driver

`bhc-driver` builds the `LinkerConfig` for native executables. Besides the compiled module objects it adds:

- `bhc_rts`: the runtime system (`bhc_init`, `bhc_rts_init`, `bhc_alloc`, ...)
- `bhc_text`, `bhc_base`, `bhc_containers`: native support libraries for `Data.Text`, `Data.Char`, `Data.Map`/`Data.Set` and similar primitives
- System libraries: `pthread`, `dl`, `m` on Linux; `System` on macOS

Library search paths come from the session's `library_paths` plus the workspace `target/debug` and `target/release` directories, where the RTS static libraries are built.
