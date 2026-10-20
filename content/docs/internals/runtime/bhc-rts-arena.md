+++
title = "bhc-rts-arena"
description = "Arena allocation"
weight = 3
+++

# bhc-rts-arena

Arena allocators for the Basel Haskell Compiler runtime.

## Overview

This crate implements the Hot Arena memory region from H26-SPEC Section 9: a bump allocator whose allocations are all freed together when the arena or a scope ends, with no garbage-collector involvement. The `frame` module adds a begin/end-frame variant for realtime loops.

**Status:** both allocators are implemented and unit-tested, and they are reachable from Rust through `bhc_rts::Runtime::run_with_arena`. Compiled Haskell does not use them yet: the crate exports no C functions, and when the native backend lowers a Loop IR buffer marked `HotArena` it emits a stack allocation instead (`crates/bhc-codegen/src/llvm/loop_lower.rs`). `bhc-numeric` has its own tensor arena (see [bhc-numeric](@/docs/internals/stdlib/bhc-numeric.md)).

This is the runtime's arena. The compiler's own arena for compile-time data is the separate `bhc-arena` crate.

## HotArena

```rust
use bhc_rts_arena::{with_arena, with_scope, HotArena};

// Arena and all its allocations are freed when the closure returns
let total = with_arena(1 << 20, |arena| {
    let x = arena.alloc(42i32)?;
    let xs = arena.alloc_slice(&[1.0f64, 2.0, 3.0])?;
    let zeros = arena.alloc_slice_zeroed::<f32>(1024)?;
    Ok::<_, bhc_rts_alloc::AllocError>(*x as f64 + xs.iter().sum::<f64>() + zeros[0] as f64)
});
```

| Method | Behavior |
|--------|----------|
| `HotArena::new(capacity)` | Allocates one backing block, cache-line aligned |
| `with_alignment(capacity, Alignment)` | Same, with a chosen base alignment |
| `with_default_capacity()` | 1 MB (`DEFAULT_ARENA_SIZE`) |
| `alloc`, `alloc_slice`, `alloc_slice_uninit`, `alloc_slice_zeroed`, `alloc_raw` | Bump the cursor; return `AllocError::ArenaExhausted` when full (the arena does not grow) |
| `used`, `remaining`, `can_alloc`, `stats` | Usage |
| `unsafe reset()` | Rewind to empty; the caller guarantees no references remain |

## Bump Allocation

```
Arena layout:
┌────────────────────────────────────┐
│ Allocated ──►   │   Free space     │
│ objects         │                  │
└────────────────────────────────────┘
                  ▲
               cursor
```

Allocation aligns the cursor and advances it, so it is O(1). Freeing is a single deallocation of the backing block when the arena drops.

## Nested Scopes

An `ArenaScope` records the cursor when it is created and rewinds to it when dropped, freeing everything allocated inside the scope while keeping earlier allocations. It borrows the arena mutably, so the borrow checker prevents references from the scope outliving the rewind.

```rust
let mut arena = HotArena::new(1 << 20);

with_scope(&mut arena, |scope| {
    let tmp = scope.alloc_slice(&[0u8; 256]);
    // ...
}); // cursor rewound here
```

## Frame Arenas (`frame`)

For work that repeats every frame or tick:

- `FrameArena`: `begin_frame()` / `end_frame()` around each frame; `end_frame` frees everything from that frame and updates `FrameStats` (frames completed, total/max/min/average bytes per frame, allocation failures). Calling them out of order panics.
- `DoubleBufferedFrameArena`: two arenas that alternate, so data from the previous frame stays readable through `previous()` for one more frame.
- `FrameGuard` and `with_frame(&mut arena, f)`: run `f` inside a frame and end it automatically.
