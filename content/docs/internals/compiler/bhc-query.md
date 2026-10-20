+++
title = "bhc-query"
description = "Incremental compilation"
weight = 61
+++

# bhc-query

Query infrastructure for incremental compilation in the Basel Haskell Compiler.

## Overview

This crate provides building blocks for a demand-driven, memoizing query system in the style of rustc's queries and salsa. It is groundwork: `bhc-driver` lists it as a dependency but does not call it, and the compilation pipeline runs its phases directly. Incremental rebuilds in practice come from separate compilation with `.bhi` interfaces (`bhc -c`), not from this crate.

## Key Types

| Item | Description |
|------|-------------|
| `Query` trait | `type Key`, `type Value`, `compute(&self, db, key)`, `name()` |
| `QueryDatabase` trait | Current revision, revision increment, in-progress marking for cycle detection, `record_dependency` |
| `QueryStorage<Q>` | Per-query memo table (a `DashMap`) with `get`, `invalidate`, `invalidate_all` |
| `SimpleDatabase` | Basic `QueryDatabase` implementation |
| `QueryRuntime<DB>` | Wraps a database; `query(&storage, &key)` |
| `Revision` | Monotonic counter; results computed at an older revision are recomputed |
| `QueryId`, `QueryState`, `MemoizedResult<V>` | Bookkeeping |
| `QueryError` | `Cycle`, `Panic`, `Cancelled` |
| `InputQuery` | Marker trait for queries whose values are set directly |

## Example

```rust
struct Double;

impl Query for Double {
    type Key = u32;
    type Value = u64;
    fn compute(&self, _db: &dyn QueryDatabase, key: &u32) -> u64 { *key as u64 * 2 }
    fn name(&self) -> &'static str { "double" }
}

let rt = QueryRuntime::new(SimpleDatabase::new());
let storage = QueryStorage::new(Double);
let v = rt.query(&storage, &21)?;   // computed and cached
rt.db.increment_revision();         // inputs changed: next get recomputes
```

## Current Limitations

- Invalidation is revision-wide: any result older than the current revision is recomputed. Recorded dependencies are not yet used to decide what is stale (`MemoizedResult::dependencies` is stored empty).
- Cycle detection reports only the query that re-entered, not the full cycle.
- There is no parallel scheduling of queries.
