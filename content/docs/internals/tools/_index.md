+++
title = "Tools"
description = "bhc command-line tools"
weight = 4
sort_by = "weight"
+++

# Tools

bhc provides several command-line tools for compilation, interactive exploration, and inspecting compiler output.

## Tool Reference

| Tool | Description |
|------|-------------|
| [bhc](@/docs/internals/tools/bhc.md) | Main compiler driver |
| [bhci](@/docs/internals/tools/bhci.md) | Interactive REPL (Core interpreter) |
| [bhi](@/docs/internals/tools/bhi.md) | IR and report inspector |
| [bhc-playground](@/docs/internals/tools/bhc-playground.md) | Browser-based interpreter |

The source tree also contains a language server, `bhc-lsp`; it builds, but it has not been verified against an editor yet.

## Quick Reference

### bhc - Compiler

```bash
# Compile to executable
bhc Main.hs -o main

# Type check only
bhc check Main.hs

# Compile with profile
bhc --profile=numeric Main.hs -o main

# Target WebAssembly
bhc --target wasm32-wasi Main.hs -o main.wasm
```

### bhci - REPL

```bash
# Start REPL
bhci

# Load a file on startup
bhci Main.hs
```

```
bhci> let y = 21
y :: Int = 21
bhci> y * 2
42
  :: Int
bhci> map (+1) [1,2,3]
[2,3,4]
  :: [Int]
```

### bhi - Inspector

```bash
# Show the Core IR of a source file
bhi ir Main.hs

# Show per-phase compile statistics
bhi stats Main.hs

# View a kernel report (JSON)
bhi kernel report.json --failures-only
```
