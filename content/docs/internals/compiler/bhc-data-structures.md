+++
title = "bhc-data-structures"
description = "Shared data structures for BHC"
weight = 5
+++

# bhc-data-structures

Shared data structures for the Basel Haskell Compiler.

## Overview

This crate re-exports and wraps commonly used collections so that the compiler uses one hashing scheme (FxHash from `rustc-hash`) consistently, and adds a few small helpers.

## Key Types

| Type | Description |
|------|-------------|
| `FxHashMap<K, V>` / `FxHashSet<T>` | `std` hash map/set with `FxHasher` |
| `FxIndexMap<K, V>` / `FxIndexSet<T>` | Insertion-ordered `indexmap` collections with `FxHasher` |
| `IndexMap`, `IndexSet` | Re-exports from `indexmap` |
| `SmallVec`, `TinyVec<T>`, `SmallVec8<T>` | Re-export of `smallvec::SmallVec`; aliases for `SmallVec<[T; 4]>` and `SmallVec<[T; 8]>` |
| `Mutex`, `RwLock` | Re-exports from `parking_lot` |
| `FxHashMapExt`, `FxHashSetExt` | Traits adding `new()` and `with_capacity()` to the Fx collections |
| `WorkQueue<T>` | FIFO work list that ignores items it has already seen |
| `FrozenMap<K, V>` | Read-only wrapper around an `FxHashMap` |
| `UnionFind` | Disjoint-set structure over `usize` elements |

## Usage

```rust
use bhc_data_structures::{FxHashMap, FxHashMapExt, SmallVec, UnionFind, WorkQueue};

let mut map: FxHashMap<Symbol, Ty> = FxHashMap::new();
map.insert(sym, ty);

let mut args: SmallVec<[Expr; 4]> = SmallVec::new();
args.push(expr);

let mut queue = WorkQueue::new();
queue.push(root);            // returns false if `root` was already queued once
while let Some(item) = queue.pop() { /* ... */ }

let mut uf = UnionFind::new(10);
uf.union(1, 2);
assert!(uf.same_set(1, 2));
```

## Notes

- FxHash is faster than SipHash for small keys but is not DoS-resistant; it is used only for compiler-internal tables.
- Graph algorithms elsewhere in the compiler use `petgraph` directly; this crate does not provide a graph type.
