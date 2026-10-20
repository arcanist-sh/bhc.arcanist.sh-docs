+++
title = "bhc-rts"
description = "Runtime system core"
weight = 1
+++

# bhc-rts

Core runtime system for the Basel Haskell Compiler.

## Overview

`bhc-rts` is the crate that compiled programs actually link against. It is built as both an `rlib` and a `staticlib`; the native backend links `libbhc_rts.a` into every executable. It has two parts:

- **`ffi.rs`**: about 270 `#[no_mangle] extern "C"` functions that generated LLVM code calls. This is the working runtime.
- **`lib.rs`**: the Rust-side `Runtime` type, which owns a `GarbageCollector` (from `bhc-rts-gc`) and a `Scheduler` (from `bhc-rts-scheduler`), plus `RuntimeConfig` and `Profile`. It re-exports the other RTS crates as `bhc_rts::alloc`, `arena`, `gc` and `scheduler`.

## C entry points (`ffi.rs`)

| Area | Examples |
|------|----------|
| Startup / exit | `bhc_rts_init`, `bhc_init`, `bhc_shutdown`, `bhc_exit` |
| Memory | `bhc_alloc`, `bhc_alloc_zeroed`, `bhc_free` |
| Thunks | `bhc_force`, `bhc_is_thunk` |
| Partial application | runtime PAP construction and application |
| Numeric primitives | `Int`, `Double`, `Float`, `Char` comparison and arithmetic; `bhc_sqrt_double`, `bhc_pow_double`, ... |
| `Integer` | `bhc_integer_*`, backed by `num-bigint` |
| `Rational` | `bhc_rational_*` (heap-allocated numerator/denominator pair) |
| Show | `bhc_show_int`, `bhc_show_double`, `bhc_show_list`, `bhc_show_maybe`, ... |
| Exceptions | `bhc_throw`, `bhc_catch`, `bhc_try_typed`, `bhc_bracket`, `bhc_finally`, masking state, `bhc_handle_uncaught_exception` |
| IO | `IORef`, `stdin`/`stdout`/`stderr`, `bhc_open_file`, handle operations, `bhc_get_args`, `bhc_get_env` |
| Directory / path | `bhc_create_directory`, `bhc_list_directory`, `bhc_remove_file`, `bhc_take_extension`, `bhc_combine`, ... |

Thunks are heap objects with an `i64` tag word: `-1` is an unevaluated thunk, `-2` is a blackhole (being evaluated; re-entering one is reported as a cycle), and a non-negative tag means the payload holds the value.

Haskell functions that the compiler knows about but does not implement are compiled to a call that aborts with `stub: <name> not implemented` when it runs.

## Startup

The generated `main` does three things:

```
main(argc, argv):
    bhc_rts_init(argc, argv)     // ignores argc/argv; calls bhc_init()
    __bhc_haskell_main(null)     // the program's Haskell main
    bhc_shutdown()               // currently a no-op
```

`bhc_init` creates the global `Runtime` with `RuntimeConfig::default()`, which is the `Default` profile. That constructs a `GarbageCollector` and a `Scheduler`; the scheduler starts one worker thread per CPU (a program blocked on `getLine` on a 10-core machine shows 11 threads). No Haskell code runs on those workers, but they poll for work with `thread::yield_now()` rather than sleeping, so they keep the cores busy while the program runs (see [bhc-rts-scheduler](@/docs/internals/runtime/bhc-rts-scheduler.md)).

`bhc_init_with_profile` also exists and maps `0..3` to `Default`, `Server`, `Numeric` and `Edge`, but generated code does not call it.

## Memory

`bhc_alloc` allocates from the system allocator with 8-byte alignment. Generated code never calls `bhc_free` and no collector runs, so heap objects live until the process exits. `bhc_gc` calls `Runtime::force_gc`, which runs the GC crate's placeholder `major_collect`; generated code does not emit it.

## Rust API (`lib.rs`)

```rust
use bhc_rts::{Profile, Runtime, RuntimeConfig};

let runtime = Runtime::new(RuntimeConfig::for_profile(Profile::Server));

let value = runtime.run(|| 42);

// Temporary allocations in a hot arena, freed when the closure returns
let doubled = runtime.run_with_arena(4096, |arena| {
    *arena.alloc(21i32).unwrap() * 2
});

// Run a closure on the work-stealing scheduler
let task = runtime.spawn(|| 6 * 7);
assert_eq!(task.await_result().unwrap(), 42);
```

There is also a process-wide runtime: `bhc_rts::init(config)`, `init_default()`, `global()`, `run(f)` and `spawn(f)`.

## Configuration

```rust
pub struct RuntimeConfig {
    pub profile: Profile,                          // default: Profile::Default
    pub gc_config: Option<GcConfig>,               // overrides the profile's GC config
    pub scheduler_config: Option<SchedulerConfig>, // overrides the profile's scheduler config
    pub debug_mode: bool,                          // default: false
    pub collect_stats: bool,                       // default: true
}
```

`Profile` has six variants: `Default`, `Server`, `Numeric`, `Edge`, `Realtime` and `Embedded`. `Profile::gc_config()` and `Profile::scheduler_config()` give each one's settings, and `uses_gc()`, `has_bounded_pauses()`, `uses_static_allocation()` and `pause_guarantee_us()` (500 µs for `Server`, 1000 µs for `Realtime`) describe the intent. See the [runtime overview](@/docs/internals/runtime/_index.md) for the table, and for which of these settings affect compiled programs today.
