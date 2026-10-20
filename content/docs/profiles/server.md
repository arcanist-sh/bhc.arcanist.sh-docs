+++
title = "Server Profile"
description = "Structured concurrency and observability for services"
weight = 2
+++

The **server** profile is reserved for long-running services: structured concurrency, cancellation, deadlines and tracing hooks. The runtime pieces for that exist and are tested in Rust. They are not yet reachable from compiled Haskell, so today a server build compiles and behaves exactly like a [default](@/docs/profiles/default.md) build.

## Usage

```bash
bhc --profile=server Server.hs -o server
```

## What it does today

- Same pipeline as the default profile: lazy evaluation, the Core simplifier, demand analysis and worker/wrapper, dictionary specialisation
- Programs are single-threaded; `forkIO` is not implemented in native code yet
- The runtime's server settings (a 500 µs GC pause target, a work-stealing scheduler) are defined in `rts/bhc-rts`, but compiled programs start the runtime with its default configuration, so they do not apply yet

## What exists in the runtime

The Rust runtime already contains the parts this profile is designed around, each with its own tests:

- A work-stealing task scheduler with scopes, cancellation, `await` with timeout, and a trace callback for observing task events (`rts/bhc-rts-scheduler`)
- Software transactional memory, implemented in Rust (`stdlib/bhc-concurrent/src/stm.rs`)
- An incremental garbage collector with a configurable pause target (`rts/bhc-rts-gc`)

## Planned Haskell API

The Haskell-facing API is drafted in `stdlib/bhc-concurrent/hs/BHC/Control/Concurrent/Scope.hs`. It is shown here to illustrate the design; these names do not resolve in today's build:

```haskell
withScope    :: (Scope -> IO a) -> IO a
spawn        :: Scope -> IO a -> IO (Task a)
await        :: Task a -> IO a
cancel       :: Task a -> IO ()
withDeadline :: Duration -> IO a -> IO (Maybe a)
```

A request handler would fetch two things concurrently and cancel both if the scope exits early:

```haskell
-- Design sketch: does not compile with the current release.
handleRequest :: Request -> IO Response
handleRequest req = withScope $ \scope -> do
    user   <- spawn scope (fetchUser (reqUserId req))
    orders <- spawn scope (fetchOrders (reqUserId req))
    Response <$> await user <*> await orders
```

The server profile becomes meaningful once compiled code is connected to the scheduler and the garbage collector. Progress is tracked on the [status page](https://arcanist.sh/bhc/status/).
