+++
title = "bhc-rts-gc"
description = "Garbage collection"
weight = 4
+++

# bhc-rts-gc

Garbage collector scaffolding for the Basel Haskell Compiler runtime.

## Overview

This crate holds the design and the first working pieces of a generational collector for the General Heap (H26-SPEC Section 9): object headers, generations, root sets, a remembered-set write barrier, pause-time accounting, and an incremental tri-color marker with an SATB write barrier.

**Status:** compiled programs do not use a garbage collector today. Generated code allocates with `bhc_alloc` in `bhc-rts`, which never frees, so programs leak memory safely until they exit; this is fine for batch work. Within this crate:

| Piece | State |
|-------|-------|
| `IncrementalMarker` (tri-color marking, SATB buffer, remark) | Implemented and unit-tested; the caller supplies a function that returns an object's children |
| Pause measurement and `PauseStats` (min/max/average/p99, threshold violations) | Implemented and unit-tested |
| `GarbageCollector::alloc` | Allocates from the system allocator with an `ObjectHeader` in front; there are no generation spaces yet |
| `minor_collect`, `major_collect` | Placeholders: they record a pause and bump a counter but do not mark, copy or free anything |
| `do_incremental_work` | Drives the marker, but with a scan function that reports no children, because objects carry no layout information yet |
| Sweep, copying, compaction, promotion | Not implemented |

Wiring a real collector into compiled code also needs the compiler to report GC roots, which it does not do today.

## Generations

```
┌──────────────────┬──────────────────┬──────────────────┐
│  Nursery (G0)    │  Survivor (G1)   │  Old (G2)        │
│  new objects     │  survived G0     │  long-lived      │
└──────────────────┴──────────────────┴──────────────────┘
┌──────────────────┐
│  Pinned region   │  never moved (FFI, DMA)
└──────────────────┘
```

`Generation` is `Nursery`, `Survivor` or `Old`, with `promote()` giving the next one. Each object has an `ObjectHeader` (atomic flag word, size, type tag); `HeaderFlags` packs the mark bit, generation and pinned bit.

## Key Types

| Item | Purpose |
|------|---------|
| `GarbageCollector` | Entry point: `alloc`, `alloc_pinned`, `minor_collect`, `major_collect`, the incremental methods below, `stats`, `pause_summary` |
| `GcConfig` | Generation sizes, promotion threshold, `incremental`, `max_pause_us` |
| `GcPtr` | A `Send + Sync` wrapper around `NonNull<u8>` |
| `GcHandle<T>` | Typed handle to a GC-managed object |
| `RootSet` | Stack, global and pinned roots |
| `WriteBarrier` | Records old objects written to (`record`) into a remembered set (`take_remembered_set`) |
| `GcStats` | Collection counts, bytes collected and promoted, GC time, max pause |
| `CollectionKind` | `Minor`, `Major`, `IncrementalMark`, `IncrementalSweep` |

## Incremental Marking (`incremental`)

Objects are white (unmarked), gray (marked, children not yet scanned) or black (marked and scanned). The marker keeps the invariant that no black object points to a white one with a snapshot-at-the-beginning barrier: when a reference is overwritten during marking, the old value goes into an `SatbBuffer` and is processed in the remark phase.

`MarkState` moves through `Idle`, `RootScanning`, `Marking`, `Remark` and `Complete`.

```rust
let gc = GarbageCollector::new(GcConfig { incremental: true, max_pause_us: 1000, ..GcConfig::default() });

gc.start_incremental_collect(&roots);   // scan roots
while gc.is_incremental_marking() {
    gc.do_incremental_work();           // one bounded increment
}
gc.finish_incremental_collect();
```

`IncrementalConfig` defaults to a 500 µs budget per increment, at most 1000 objects per increment, stop-the-world increments (`concurrent: false`) and an SATB threshold of 1024 entries. `GarbageCollector::new` gives each increment half of `max_pause_us`.

## Configuration

```rust
GcConfig {
    nursery_size: 4 * 1024 * 1024,   // 4 MB
    survivor_size: 2 * 1024 * 1024,  // 2 MB
    old_gen_size: 64 * 1024 * 1024,  // 64 MB
    nursery_threshold: 2,            // collections before promotion
    incremental: false,
    max_pause_us: 1000,
}
```

These are the defaults. `bhc_rts::Profile::gc_config()` derives per-profile variants (for example `Server` turns on incremental mode with a 500 µs target).
