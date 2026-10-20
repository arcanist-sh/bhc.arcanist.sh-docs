+++
title = "bhc-playground"
description = "Web playground"
weight = 4
+++

# bhc-playground

Browser-based playground for experimenting with the Basel Haskell Compiler.

## Overview

`bhc-playground` is a crate (`crates/bhc-playground`) that packages bhc's front end and Core interpreter for the browser. It is compiled to WebAssembly (`wasm32-unknown-unknown`) with `wasm-bindgen`, and the [online playground](https://arcanist.sh/bhc/playground/) loads it to parse, type check and evaluate code locally, with no server round-trips. It does not run native code generation; programs are evaluated by the same Core interpreter as [bhci](@/docs/internals/tools/bhci.md).

## Features

| Feature | Description |
|---------|-------------|
| Run | Evaluate `main` and show its value |
| Check | Parse and type check without running |
| Profiles | Default (lazy) or Numeric (strict) evaluation |
| Error reporting | Errors are reported with a category (`parse`, `type`, `runtime`, `no_main`, ...) |
| Examples | A row of one-click example programs |
| Keyboard | Ctrl/Cmd + Enter runs the code |

The page uses a plain text editor with light syntax highlighting. There is no code sharing by URL and no embeddable version.

## Architecture

```
┌─────────────────────────────────────────┐
│ Browser                                 │
│  ┌───────────────────────────────────┐  │
│  │ Editor (textarea)                 │  │
│  └───────────────────────────────────┘  │
│                 │ source                │
│                 ▼                       │
│  ┌───────────────────────────────────┐  │
│  │ bhc_playground.wasm               │  │
│  │ parse → lower → type check →      │  │
│  │ Core → Core interpreter           │  │
│  └───────────────────────────────────┘  │
│                 │ JSON result           │
│                 ▼                       │
│  ┌───────────────────────────────────┐  │
│  │ Output                            │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

## JavaScript API

The generated module (`bhc_playground.js` + `bhc_playground_bg.wasm`) exports three functions. Each takes Haskell source text and returns a JSON string.

| Function | Description |
|----------|-------------|
| `run_haskell(source)` | Evaluate `main` with the default profile |
| `run_haskell_numeric(source)` | Evaluate `main` with the numeric profile |
| `check_haskell(source)` | Parse and type check only |

```javascript
import init, { run_haskell } from './bhc_playground.js';

await init();
const result = JSON.parse(run_haskell('main = 2 + 3 * 4'));
// { success: true,
//   output: { display: "14", result_type: null, is_function: false },
//   error: null }
```

On failure, `success` is `false` and `error` holds `{ message, category, line, column }`. `line` and `column` are not filled in yet, and `result_type` is not populated by `run_haskell`.

## Building

```bash
rustup target add wasm32-unknown-unknown
cargo install wasm-bindgen-cli --version 0.2.125

cargo build -p bhc-playground --target wasm32-unknown-unknown --release
wasm-bindgen --out-dir playground-out --target web \
  target/wasm32-unknown-unknown/release/bhc_playground.wasm
```

The `playground.yml` workflow in the bhc repository runs these steps when the playground or any crate it depends on changes, and copies the two output files to `static/bhc/playground/` in the arcanist.sh website.
