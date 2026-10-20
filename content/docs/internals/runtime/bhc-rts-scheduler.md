+++
title = "bhc-rts-scheduler"
description = "Thread scheduling"
weight = 5
+++

# bhc-rts-scheduler

Work-stealing task scheduler for the Basel Haskell Compiler runtime.

## Overview

This crate implements the structured-concurrency model of H26-SPEC Section 10 for Rust closures: a pool of OS worker threads with work-stealing deques (from `crossbeam`), scopes that wait for their tasks, cooperative cancellation, deadlines, and trace events. The integration tests in `tests/m5_exit_criteria.rs` cover scope completion, deadlines, tracing and cooperative cancellation.

**Status:** the scheduler works and is unit- and integration-tested from Rust. Compiled Haskell does not run on it: the crate exports no C functions, and Haskell's `forkIO`, `MVar` and STM operations currently compile to stubs that abort with `stub: ... not implemented` on the native backend.

The global `Runtime` that every compiled program creates at startup does construct a `Scheduler`, so a compiled program starts one worker thread per CPU. Those workers have no tasks; an idle worker polls the queues in a loop with `thread::yield_now()`, so while a compiled program is running or blocked (for example in `getLine`), the workers keep the CPU cores busy. On a 10-core machine, a program waiting on stdin showed about 550% CPU in `ps`.

## Architecture

```
┌──────────────────────────────────────────────────────┐
│ Scheduler                                            │
│   global Injector queue  ◄── spawn()                 │
├────────────┬────────────┬────────────┬───────────────┤
│ worker 0   │ worker 1   │ worker 2   │ worker N-1    │
│ local FIFO │ local FIFO │ local FIFO │ local FIFO    │
│ deque      │ deque      │ deque      │ deque         │
└────────────┴────────────┴────────────┴───────────────┘
        ◄────────── steal from each other ──────────►
```

Each worker thread is named `bhc-worker-<id>` and loops: pop from its own deque, then take from the global injector, then try to steal from every other worker's `Stealer`.

## API

```rust
use bhc_rts_scheduler::{check_cancelled, with_deadline, with_scope, Scheduler};
use std::time::Duration;

let scheduler = Scheduler::new(4);           // 4 worker threads

let sum = with_scope(&scheduler, |scope| {
    let a = scope.spawn(|| 20);
    let b = scope.spawn(|| 22);
    a.await_result().unwrap() + b.await_result().unwrap()
});                                          // scope waits for all its tasks

let result = with_deadline(&scheduler, Duration::from_secs(5), |scope| {
    scope.spawn(|| {
        while !check_cancelled() { /* work in chunks */ }
    })
});                                          // None if the deadline passed
```

| Item | Purpose |
|------|---------|
| `Scheduler::new(n)`, `with_config`, `with_default_config` | Start the worker pool |
| `Scheduler::spawn` | Run a closure outside any scope |
| `Task<T>` | `await_result`, `await_timeout`, `try_result`, `cancel`, `state`, `is_done` |
| `TaskResult<T>` | `Ok(T)`, `Cancelled`, `Panicked(..)` |
| `Scope` | `spawn`, `cancel`, `is_cancelled`; dropping a scope waits for its tasks |
| `with_scope`, `with_deadline` | Run a closure with a fresh scope; the deadline version cancels the scope when time runs out |
| `check_cancelled`, `current_task_id`, `current_worker_id` | Called from inside a task |
| `set_trace_callback`, `clear_trace_callback` | Receive `TraceEvent`s |
| `SchedulerStats` | Spawned, completed, cancelled and failed tasks; steals and failed steals |

## Task States

| State | Description |
|-------|-------------|
| `Pending` | Created, not yet running |
| `Running` | Executing on a worker |
| `Cancelling` | Cancellation requested; the task stops at its next `check_cancelled` |
| `Completed` | Finished with a value |
| `Cancelled` | Stopped by cancellation |
| `Failed` | The closure panicked |

Cancellation is cooperative: a task stops when it calls `check_cancelled()` and sees the flag.

## Trace Events

`TaskSpawn`, `TaskStart`, `TaskComplete`, `TaskCancel`, `ScopeCreate`, `ScopeExit`, `DeadlineSet` and `DeadlineReached`, each with ids and a timestamp.

## Configuration

```rust
SchedulerConfig {
    num_workers: available_parallelism(),  // default: one per CPU
    stack_size: 2 * 1024 * 1024,           // 2 MB per worker
    work_stealing: true,
}
```

There are no Haskell sparks: `par`/`pseq`-style speculative parallelism is not part of this scheduler.
