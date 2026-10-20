+++
title = "bhc-arena"
description = "Arena memory allocation for BHC"
weight = 2
+++

# bhc-arena

Arena-based memory allocation for the Basel Haskell Compiler.

## Overview

This crate wraps [`bumpalo`](https://docs.rs/bumpalo) and [`typed-arena`](https://docs.rs/typed-arena) behind a small set of allocator types for compiler data structures. Memory is released in bulk when the arena is dropped.

Several crates (`bhc-ast`, `bhc-hir`, `bhc-core`, `bhc-types`, `bhc-tensor-ir`, `bhc-loop-ir`) list `bhc-arena` as a dependency, but none of them currently import it in their source; the IRs are built from ordinary owned Rust values (`Box`, `Vec`, `Arc`).

## Key Types

| Type | Description |
|------|-------------|
| `Arena` | Single-threaded bump arena (`bumpalo::Bump`) that also counts bytes allocated |
| `DroplessArena` | Bump arena for values that do not need destructors |
| `SyncArena` | Bump arena behind a `parking_lot::Mutex`, shareable across threads |
| `Bump` | Re-export of `bumpalo::Bump` |
| `TypedArena<T>` | Re-export of `typed_arena::Arena<T>` (runs destructors) |

## Usage

```rust
use bhc_arena::Arena;

let arena = Arena::new();            // or Arena::with_capacity(n)

let x: &mut i32 = arena.alloc(42);
let xs: &mut [u32] = arena.alloc_slice(&[1, 2, 3]);
let s: &str = arena.alloc_str("hello");
let ys: &mut [usize] = arena.alloc_from_iter(0..5);

assert_eq!(arena.bytes_allocated(), /* sum of the sizes above */);
// arena.allocated_bytes_including_metadata() reports bumpalo's chunk usage.
// `unsafe fn reset(&mut self)` frees everything and invalidates references.
```

## Notes

- `Arena`, `DroplessArena` and `SyncArena` are all backed by `bumpalo`, which does not run destructors for allocated values. Use `TypedArena<T>` when values own heap memory that must be dropped.
- `SyncArena::alloc` requires `T: Send` and returns `&T` (not `&mut T`).
