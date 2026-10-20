+++
title = "bhc-macros"
description = "Procedural macros"
weight = 66
+++

# bhc-macros

Procedural macros for the Basel Haskell Compiler.

## Overview

This is a `proc-macro` crate (dependencies: `syn`, `quote`, `proc-macro2`) intended to remove boilerplate from compiler data structures. It is currently scaffolding: no other crate in the workspace depends on it, and several macros are placeholders. Interning, spans and index types are implemented by hand in `bhc-intern`, `bhc-span` and `bhc-index`.

## Available Macros

| Macro | Kind | What it generates today |
|-------|------|-------------------------|
| `Internable` | derive | For a `struct T(String)`: `new`, `as_str`, `Display`, `AsRef<str>`, `From<String>`, `From<&str>`, `Borrow<str>` |
| `AstNode` | derive | A test-only `dummy()` (via `Default`) and a `Debug` impl that prints only the type name |
| `IrNode` | derive | Stub `is_normal_form()` (always `true`) and `size()` (always `1`) |
| `EnumDispatch` | derive | A trivial `map(&self, f)` that applies `f` to `self` |
| `#[query]` | attribute | Wraps the function body with `tracing::trace!` calls in debug builds; no memoization |
| `error_codes!` | function-like | Ignores its input; emits a fixed `error_codes` module with `E0001`/`E0002` |
| `define_tokens!` | function-like | Ignores its input; emits a `TokenKind` enum with only `Eof` and `Error` |
| `impl_visitor!` | function-like | Placeholder; expands to nothing |

## Usage

```rust
use bhc_macros::Internable;

#[derive(Internable)]
pub struct Identifier(String);

let id = Identifier::new("main");
assert_eq!(id.as_str(), "main");
```

`Internable` is the only macro whose output is complete. Despite the name, it does not intern: it adds string conversions to a `String` newtype. Real interning lives in `bhc-intern`.
