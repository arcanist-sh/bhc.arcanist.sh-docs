+++
title = "Runtime Crates"
description = "bhc Runtime System crates"
weight = 2
sort_by = "weight"
+++

# Runtime Crates

The bhc Runtime System (RTS) is the Rust code that compiled Haskell programs link against. It lives under `rts/` in the compiler repository as five crates. One of them, `bhc-rts`, is built as a static library (`libbhc_rts.a`) and is what the native backend links into every executable; the other four are libraries that `bhc-rts` depends on.

## What a compiled program uses today

A native executable produced by `bhc` links `libbhc_rts.a` together with `libbhc_text.a`, `libbhc_base.a` and `libbhc_containers.a` (see the driver's link step in `crates/bhc-driver/src/lib.rs`). At run time it uses:

- **The C entry points in `bhc-rts`** (`rts/bhc-rts/src/ffi.rs`): heap allocation (`bhc_alloc`), thunk forcing, partial applications, `Integer` and `Rational` arithmetic, `Double`/`Float` math, `show` helpers, exceptions, `IORef`, handles and file IO, directory and environment access, and process exit.
- **A leak allocator.** `bhc_alloc` is a thin wrapper over the system allocator. Generated code never frees heap objects and no collector runs, so a program's memory grows until it exits. This is safe and works well for batch programs such as compilers, converters and test runs; it is not yet suitable for long-running services.

The memory-management and scheduling crates are real, unit-tested Rust code, but compiled Haskell does not call into them yet:

| Crate | Implemented | Reached from compiled Haskell |
|-------|-------------|-------------------------------|
| [bhc-rts](@/docs/internals/runtime/bhc-rts.md) | C entry points, `Runtime`, profiles | Yes: the C entry points are the runtime |
| [bhc-rts-alloc](@/docs/internals/runtime/bhc-rts-alloc.md) | Alignment, pinned allocator, static (no-GC) allocator | No |
| [bhc-rts-arena](@/docs/internals/runtime/bhc-rts-arena.md) | Hot arena, frame arena | No |
| [bhc-rts-gc](@/docs/internals/runtime/bhc-rts-gc.md) | GC data structures, incremental tri-color marker | No |
| [bhc-rts-scheduler](@/docs/internals/runtime/bhc-rts-scheduler.md) | Work-stealing scheduler, structured concurrency | No (its worker threads start and poll for work, but run no Haskell code) |

## Architecture

```
┌──────────────────────────────────────────────────────┐
│        Compiled bhc program (LLVM object code)       │
└──────────────────────────┬───────────────────────────┘
                           │ calls bhc_* C functions
┌──────────────────────────▼───────────────────────────┐
│  bhc-rts  (libbhc_rts.a)                             │
│  ffi.rs: bhc_rts_init, bhc_alloc, bhc_force, IO,     │
│          Integer, exceptions, show, ...              │
│  lib.rs: Runtime, RuntimeConfig, Profile             │
└───────┬──────────────┬──────────────┬────────────────┘
        │              │              │
┌───────▼──────┐ ┌─────▼──────┐ ┌─────▼─────────────┐
│ bhc-rts-gc   │ │ bhc-rts-   │ │ bhc-rts-scheduler │
│              │ │ arena      │ │                   │
└───────┬──────┘ └─────┬──────┘ └─────┬─────────────┘
        └──────────────┼──────────────┘
               ┌───────▼───────┐
               │ bhc-rts-alloc │
               └───────────────┘
```

## Memory Regions

The design (H26-SPEC Section 9) has three host memory regions. Each has a Rust implementation; only the general heap is used by compiled programs, and only as a leak allocator.

| Region | Purpose | Implementation | Used by compiled code |
|--------|---------|----------------|-----------------------|
| **Hot Arena** | Kernel temporaries | `bhc_rts_arena::HotArena`, bump pointer | No |
| **Pinned Heap** | FFI / device IO | `bhc_rts_alloc::PinnedAllocator`, `bhc_pinned_alloc` | No |
| **General Heap** | Boxed values | `bhc_alloc` (system allocator, never freed) | Yes |

`bhc-rts-alloc` also defines a `DeviceMemory` region kind for GPU memory.

## Profiles

`bhc_rts::Profile` has six variants, each with a GC and scheduler configuration: `Default`, `Server`, `Numeric`, `Edge`, `Realtime` and `Embedded`.

| Profile | GC config | Scheduler config |
|---------|-----------|------------------|
| Default | 4 MB nursery, not incremental | one worker per CPU, work stealing |
| Server | incremental, 500 µs pause target | work stealing |
| Numeric | 1 MB nursery | work stealing |
| Edge | small generations | 1 worker, no stealing |
| Realtime | incremental, 1 ms pause target | 2 workers, no stealing |
| Embedded | GC disabled, static allocation | 1 worker, 64 KB stack |

The compiler's `--profile` flag changes what the compiler does (for example, strictness and the Tensor IR path for `numeric`). It does not change the runtime: the generated `main` calls `bhc_rts_init`, which ignores its arguments and initializes the `Default` profile. Because no collector runs, the GC settings above have no effect on compiled programs today.
