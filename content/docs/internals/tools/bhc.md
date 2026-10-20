+++
title = "bhc"
description = "Main compiler"
weight = 1
+++

# bhc

The main Basel Haskell Compiler command-line interface.

## Overview

`bhc` is the primary compiler executable. It runs every phase from source to executable: parsing, lowering, type checking, Core, code generation through LLVM (or the WebAssembly backend), and linking.

## Basic Usage

```bash
# Compile a single file (follows its imports)
bhc Main.hs -o main

# Compile with a specific optimization level
bhc -O3 Main.hs -o main

# Type check only
bhc check Main.hs

# Compile to an object file without linking
bhc -c Main.hs --odir build --hidir build
```

## Subcommands

| Subcommand | Description |
|------------|-------------|
| `bhc FILE...` | Compile and link (same as `bhc build`) |
| `bhc build FILE...` | Compile and link |
| `bhc check FILE...` | Parse and type check without generating code; accepts files or directories |
| `bhc run FILE` | Evaluate a program with the Core interpreter (no native code) |
| `bhc repl` | Start the interactive REPL ([bhci](@/docs/internals/tools/bhci.md)) |
| `bhc version` | Show version information |
| `bhc completions SHELL` | Generate shell completions (bash, zsh, fish, ...) |

Options such as `-o` belong before the subcommand: `bhc -o main build Main.hs`. The bare form `bhc Main.hs -o main` takes them in any position.

`bhc run` interprets the program rather than compiling it, does not pass command-line arguments through yet, and currently prints the program's output twice. To run a program, compile it and execute the binary.

## Common Options

| Flag | Description |
|------|-------------|
| `-o, --output FILE` | Output file name (defaults to the source file's name without `.hs`) |
| `-O, --opt-level N` | Optimization level 0-3 (default 2) |
| `--profile NAME` | Runtime profile: `default`, `server`, `numeric`, `edge` |
| `--target TRIPLE` | Target, e.g. `wasm32-wasi`; see [targets](@/docs/targets/_index.md) |
| `-c, --compile-only` | Compile to an object file only, no link |
| `--odir DIR` / `--hidir DIR` | Where `-c` writes object files / `.bhi` interfaces |
| `-I, --import-path DIR` | Additional directory to search for imported modules |
| `--package-dir DIR` | Dependency source root used to resolve imports during `check` |
| `--package-db PATH` | Directory of compiled `.bhi` interfaces to resolve imports from |
| `--package-id ID` | Expose a package from the package database |
| `-D MACRO[=VALUE]` | CPP define |
| `--kernel-report` | Print the numeric kernel report to stderr |
| `-v` / `-vv` / `-vvv` | More verbose logging |
| `-q, --quiet` | Suppress non-error output |
| `-h, --help` | Show help |

`--edition`, `--dump-ir`, `-j/--jobs` and `--emit` (other than `--emit ptx`) are accepted on the command line but have no effect yet.

## Target Selection

```bash
# Native, for the host (default)
bhc Main.hs -o main

# WebAssembly (WASI)
bhc --target wasm32-wasi Main.hs -o main.wasm

# Object file for another architecture (linking for another platform is not supported yet)
bhc --target x86_64-unknown-linux-gnu -c Main.hs --odir build
```

## Packages

`bhc` itself compiles files and modules; project management (creating projects, resolving and fetching dependencies, running test suites) is the job of [hx](https://arcanist.sh/hx/). `bhc` has no `init`, `add` or `test` subcommands; it provides `-c`, `--package-db` and `--package-id` as the building blocks for package builds.
