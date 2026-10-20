+++
title = "bhc-containers"
description = "Data structures"
weight = 3
+++

# bhc-containers

Container data structures for the Basel Haskell Compiler.

## Overview

This crate implements `Data.Map`, `Data.Set`, `Data.IntMap`, `Data.IntSet` and `Data.Sequence` in Rust. It is built as `libbhc_containers.a`, which the native backend links into every program; the compiler lowers container operations to its `bhc_map_*`, `bhc_set_*`, `bhc_intmap_*` and `bhc_seq_*` functions. The `hs/BHC/Data/{Map, Set, IntMap, IntSet, Sequence}.hs` sources are not compiled into programs.

## Representation

| Type | Rust representation | Notes |
|------|---------------------|-------|
| `Map k v` | `BTreeMap<i64, *mut u8>` behind a pointer | Keys are compared as 64-bit words |
| `Set a` | `BTreeSet<i64>` behind a pointer | Same key handling as `Map` |
| `IntMap v` | Same as `Map` | The `bhc_intmap_*` functions share the `Map` code |
| `IntSet` | `BTreeSet<i64>` | Own `bhc_intset_*` functions |
| `Seq a` | An ordinary cons list, same layout as `[a]` | `fromList` and `toList` are the identity |

**Keys.** `Int` and other unboxed keys are stored as their value, so they order exactly as in GHC. Boxed keys such as `String` and `Text` are canonicalized by content (`bhc_map_canon` and `bhc_set_canon` intern each distinct key and store the pointer of its first occurrence), so lookup, membership and `insertWith` see equal strings as the same key. The order of boxed keys is the order in which each key was first seen, not lexicographic order: `Map.toList (Map.fromList [("b",1),("a",2)])` gives `[("b",1),("a",2)]`, where GHC gives `[("a",2),("b",1)]`.

**Persistence.** Maps and sets are immutable from Haskell's point of view: `insert`, `delete`, `union`, `intersection` and `difference` clone the whole tree and return a new one.

**Seq.** Keeping `Seq` list-shaped lets generic `Foldable` code, and libraries such as pandoc's `Builder` (`Many (Seq a)`), traverse a `Seq` like a list.

## Complexity

| Operation | Map / Set / IntMap / IntSet | Seq |
|-----------|-----------------------------|-----|
| `lookup`, `member`, `findWithDefault` | O(log n) | |
| `insert`, `delete`, `adjust` | O(n) (copy) + O(log n) | |
| `union`, `intersection`, `difference` | O(n) copy + O(m log n) | |
| `size` | O(1) | |
| `<\|` (cons) | | O(1) |
| `\|>`, `><`, `index`, `update`, `length` | | O(n) |

## What works today

Checked against GHC 9.8.2 on 2026-10-20, one expression per program:

- `Data.Map`: `empty`, `singleton`, `fromList`, `fromListWith`, `toList`, `insert`, `insertWith`, `delete`, `adjust`, `lookup` (including `String` keys), `member`, `findWithDefault`, `size`, `keys`, `elems`, `map`, `filter`, `foldr`, `foldrWithKey`, `mapWithKey`, `union`, `unionWith`
- `Data.Set`: `fromList`, `toList`, `member`, `size`, `union`
- `Data.IntMap`: `fromList`, `size`, `findWithDefault`
- `Data.IntSet`: `fromList`, `toList`, `member`
- `Data.Sequence`: `fromList`, `<|`, `|>`, `length`, `index`, `update`, `sum` via `Foldable`

## Known gaps

- `Data.Map.Strict` is not implemented: every operation imported from it (`fromList`, `toList`, `lookup`, ...) aborts with `stub: Data.Map.Strict.<name> not implemented`. Import `Data.Map` instead.
- Boxed keys come back in first-insertion order, as described above.
- `Map.alter (fmap (+1)) 1 m` stores `0` instead of the updated value.
- `Map.lookupMin` fails to compile; `Set.map` fails to type-check.
- `print (IntMap.toList m)` and `print (IntMap.lookup k m)` print a pointer.
- `show` of a `Seq` omits GHC's `fromList` prefix: `[10,99,30]` instead of `fromList [10,99,30]`.
- This crate has no graph or tree types, and `Data.Graph` is not available.
