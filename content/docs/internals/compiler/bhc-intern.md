+++
title = "bhc-intern"
description = "String interning for BHC"
weight = 3
+++

# bhc-intern

String interning for the Basel Haskell Compiler.

## Overview

This crate stores each unique string once in a single process-wide interner and hands out `Symbol`s, which are `u32` indices into it. Symbol equality and hashing are integer operations. Interned strings are leaked, so `Symbol::as_str` returns `&'static str`.

The interner itself is private: there is no `Interner` value to create or pass around. All interning goes through `Symbol::intern`.

## Key Types

| Type | Description |
|------|-------------|
| `Symbol` | An interned string (`#[repr(transparent)]` over `u32`) |
| `Ident` | An identifier wrapping a `name: Symbol` |
| `kw` | Module of pre-interned keyword and common-name symbols |

## Usage

```rust
use bhc_intern::{Ident, Symbol};

let a = Symbol::intern("hello");
let b: Symbol = "hello".into();     // From<&str> / From<String>

assert_eq!(a, b);                   // u32 comparison
assert_eq!(a.as_str(), "hello");    // &'static str
assert!(a == "hello");              // PartialEq<&str>

let x = Ident::from_str("x");       // or Ident::new(sym)
```

`Ord` on `Symbol` compares the underlying strings, not the indices, so sorted output is stable across runs.

## Pre-interned Keywords

The `kw` entries are `LazyLock<Symbol>` statics, so they are dereferenced to get a `Symbol`:

```rust
use bhc_intern::kw;

kw::intern_all();          // optional: intern every keyword up front

let if_kw: Symbol = *kw::IF;
let where_kw = *kw::WHERE;
let just = *kw::JUST;
```

The set covers Haskell keywords, the H26 keywords `lazy`, `strict`, `profile` and `edition`, common type names (`Int`, `Double`, `String`, ...) and common constructors (`True`, `Just`, `Left`, ...).

## Serialization

A symbol's index is only meaningful within one process. `Symbol` therefore serializes as its string and re-interns on deserialization, which lets serialized Core (the `.bhc` cross-module body sidecars) and interface files round-trip between separate compiler invocations.
