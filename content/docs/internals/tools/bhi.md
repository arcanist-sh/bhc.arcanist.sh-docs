+++
title = "bhi"
description = "IR and report inspector"
weight = 3
+++

# bhi

The Basel Haskell Inspector: a diagnostic viewer for bhc's intermediate representations and compiler reports.

## Overview

`bhi` shows what the compiler does with a program. Given a Haskell source file, it runs the front end itself and prints the requested IR stage or per-phase statistics. Its other subcommands read JSON reports (kernel fusion, memory, call graph, compile statistics).

`bhi` is not a bytecode interpreter; bhc has no bytecode format. To evaluate code without compiling it, use [bhci](@/docs/internals/tools/bhci.md) or `bhc run`.

## Usage

```bash
# Core IR of a source file (default stage)
bhi ir Main.hs

# Another stage: ast, hir, core, tensor, loop
bhi ir --stage hir Main.hs

# Only lines that mention a name
bhi ir --function square Main.hs

# Lines, declarations, Core bindings and time per phase
bhi stats Main.hs
```

```
$ bhi ir Square.hs
IR Inspection (from source)
File: Square.hs
Stage: Core

Core IR
────────────────────────────────────────────────────────────
module Main where

square = \arg0_200028 -> * arg0_200028 arg0_200028
main = print (square 7)
```

## Subcommands

| Subcommand | Input | Description |
|------------|-------|-------------|
| `ir` | `.hs` source, or an IR dump | Show an IR stage (`--stage`, `--function`) |
| `stats` | `.hs` source, or stats JSON | Per-phase compile statistics (`--timing`, `--compare`) |
| `kernel` | kernel report JSON | Fusion report (`--failures-only`, `--timing`, `--simd`, `--format text\|json\|dot`) |
| `memory` | allocation report JSON | Allocation summary (`--heap-only`, `--arena`, `--by-site`) |
| `callgraph` | call graph JSON | Call graph (`--filter`, `--depth`, `--cycles`, `--format`) |
| `diff` | two IR dumps | Compare two dumps (`--changes-only`, `--context N`) |
| `pretty` | IR dump | Re-indent an IR file (`--width`, `--indent`) |
| `validate` | IR dump | Check an IR file (`--types`, `--invariants`) |

## Current Status

The source-file paths of `ir` and `stats` run the real front end. The report-reading subcommands are ahead of the compiler: `bhc` does not write these JSON files yet (`--dump-ir` is accepted but unused, and `--kernel-report` prints a text report to stderr). When `kernel`, `memory`, `callgraph` or `stats` are given a file that is not in their JSON format, they display built-in sample data rather than an error, so treat their output as a demonstration of the format until the compiler produces the files. `bhi` has unit tests but no integration tests yet.
