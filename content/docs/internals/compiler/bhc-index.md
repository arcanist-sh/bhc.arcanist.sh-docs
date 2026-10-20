+++
title = "bhc-index"
description = "Type-safe indices for BHC"
weight = 4
+++

# bhc-index

Type-safe index types for the Basel Haskell Compiler.

## Overview

This crate provides strongly-typed index newtypes so that, for example, an expression index cannot be used where a type index is expected, plus a vector that can only be indexed by its own index type.

## Key Items

| Item | Description |
|------|-------------|
| `Idx` | Trait for index types: `fn new(usize) -> Self` and `fn index(self) -> usize` |
| `define_index!` | Macro that declares `u32` newtype indices implementing `Idx` |
| `IndexVec<I, T>` | `Vec<T>` indexed by `I: Idx` |
| `IndexMap<I, T>` | Alias for `IndexVec<I, Option<T>>` |

## Usage

```rust
use bhc_index::{define_index, IndexVec};

define_index! {
    /// Index into the expression arena.
    pub struct ExprId;

    /// Index into the type arena.
    pub struct TypeId;
}

let e = ExprId::new(0);            // inherent const fn new(u32)
let t = TypeId::from_usize(0);
// let wrong: ExprId = t;          // does not compile

let mut exprs: IndexVec<ExprId, String> = IndexVec::new();
let id = exprs.push("x".to_string());
assert_eq!(exprs[id], "x");

for (id, e) in exprs.iter_enumerated() { /* ... */ }
let next = exprs.next_index();
```

Each generated type is `#[repr(transparent)]` over `u32`, derives `Copy`, `Eq`, `Ord` and `Hash`, has `as_u32`/`as_usize`, `From` conversions to and from `u32`/`usize`, and serializes as a plain integer. The macro expansion refers to `serde::Serialize`, so the defining crate needs `serde` as a dependency.

`IndexVec` also offers `with_capacity`, `from_vec`, `get`/`get_mut`, `iter`/`iter_mut`, `indices`, and `raw`/`raw_mut`/`into_raw` for access to the underlying `Vec`.

## Benefits

- Compile-time prevention of ID mixups
- No runtime overhead (newtypes over `u32`)
- Indexed collections that serialize transparently
