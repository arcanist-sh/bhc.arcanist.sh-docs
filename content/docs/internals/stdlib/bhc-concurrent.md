+++
title = "bhc-concurrent"
description = "Concurrency primitives"
weight = 7
+++

# bhc-concurrent

Concurrency primitives for the Basel Haskell Compiler.

## Overview

This crate implements software transactional memory, structured-concurrency scopes, tasks and channels in Rust, with a C interface in `ffi.rs` for compiled code to call. The intended Haskell API is in `hs/BHC/Control/Concurrent/Scope.hs` and `hs/BHC/Control/Concurrent/STM.hs`.

**Status:** the Rust implementation is complete enough to be unit-tested, but it is not yet connected to compiled Haskell. `libbhc_concurrent.a` is not on the native link line, and generated code calls none of its functions. In a compiled program today, `forkIO`, `newMVar`, `takeMVar`, `atomically`, `newTVar` and the rest of `Control.Concurrent` and `Control.Concurrent.STM` type-check and compile, then abort when run with `stub: <name> not implemented` (checked 2026-10-20 with `forkIO`/`MVar` and `atomically`/`TVar` programs that GHC runs).

This crate is also independent of [bhc-rts-scheduler](@/docs/internals/runtime/bhc-rts-scheduler.md): its scopes run each task on its own `std::thread`.

## Rust modules

| Module | Contents |
|--------|----------|
| `stm` | `TVar<T>` with transactional `read_tx`/`write_tx`; `atomically`, `retry`, `or_else`, `check`, `abort`; `TMVar`, `TChan`, `TQueue` |
| `scope` | `with_scope`, `with_deadline`, `spawn`, `spawn_all`, `SpawnHandle` (`join`, `try_join`, `cancel`, `state`), `check_cancelled`, `checkpoint`; child scopes inherit cancellation |
| `task` | `Task<T>`, `TaskState`, `TaskFuture` (`poll`, `wait`, `wait_timeout`), cooperative cancellation |
| `channel` | `BoundedChannel<T>` and `UnboundedChannel<T>` over `crossbeam-channel` |
| `ffi` | The C interface below |

```rust
use bhc_concurrent::stm::{atomically, TVar};

let a = TVar::new(100);
let b = TVar::new(200);

atomically(|| {
    let x = a.read_tx()?;
    let y = b.read_tx()?;
    a.write_tx(x - 50)?;
    b.write_tx(y + 50)?;
    Ok(())
});
```

## C interface (`ffi.rs`)

| Area | Functions |
|------|-----------|
| Scopes and tasks | `bhc_with_scope`, `bhc_with_deadline`, `bhc_spawn`, `bhc_await`, `bhc_poll`, `bhc_cancel`, `bhc_task_free`, `bhc_check_cancelled` |
| STM | `bhc_atomically`, `bhc_retry`, `bhc_check`; `bhc_tvar_{new,read,write,free}_i64` and `_f64` |
| TMVar / TQueue | `bhc_tmvar_{new,new_empty,take,put,free}_i64`; `bhc_tqueue_{new,write,read,try_read,is_empty,free}_i64` |
| Misc | `bhc_num_cpus` |

The C-level transactional variables hold `i64` or `f64` values; boxed Haskell values are not supported at this layer yet.

## Not provided

There is no `MVar`, `Chan`, `async`-style API or thread-delay primitive in this crate. In compiled programs, `newMVar`, `writeChan` and `threadDelay` abort as stubs; `myThreadId` returns.
