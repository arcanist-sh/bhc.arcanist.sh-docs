+++
title = "bhc-transformers"
description = "Monad transformers"
weight = 6
+++

# bhc-transformers

Monad transformers for the Basel Haskell Compiler.

## Overview

The Rust side of this crate is intentionally empty: transformers need no runtime primitives. What programs use today is the compiler's **built-in** support for `transformers` and `mtl`: `bhc-lower` maps the standard module names to builtin definitions, `bhc-typeck` knows `StateT`, `ReaderT`, `WriterT`, `ExceptT` and `Identity` and the `MonadState`, `MonadReader`, `MonadWriter` and `MonadError` classes, and `bhc-codegen` implements the common stack shapes directly (`spec/BHC-BRIEF-0005` lists every stage involved).

The `hs/` directory holds ordinary Haskell implementations of the same modules, under both `BHC.Control.Monad.*` and the standard names (`Control.Monad.State`, `Control.Monad.Trans.State.Strict`, `Control.Monad.Trans.Maybe`, `Data.Functor.Identity`, ...). Moving to them is the plan in BHC-BRIEF-0005: with `BHC_IO_MODEL=closure` set, the driver puts `stdlib/bhc-transformers/hs` on the module search path (or the directory in `BHC_TRANSFORMERS_PATH`) and compiles the transformers like a user library. That mode is experimental; the built-in path is the default.

## Modules

The names below are in the compiler's export tables; the next section lists which have been checked end to end.

| Module | Provides |
|--------|----------|
| `Control.Monad.State`, `.Strict`, `Control.Monad.Trans.State` | `State`, `StateT`, `get`, `put`, `modify`, `gets`, `runState`, `evalState`, `execState`, `runStateT`, ... |
| `Control.Monad.Reader`, `Control.Monad.Trans.Reader` | `Reader`, `ReaderT`, `ask`, `asks`, `local`, `runReader`, `runReaderT` |
| `Control.Monad.Writer`, `.Strict`, `Control.Monad.Trans.Writer` | `Writer`, `WriterT`, `tell`, `runWriter`, `execWriter`, `runWriterT` |
| `Control.Monad.Except`, `Control.Monad.Trans.Except` | `Except`, `ExceptT`, `throwError`, `catchError`, `runExcept`, `runExceptT` |
| `Control.Monad.RWS`, `Control.Monad.Trans.RWS.Strict` | `RWS`, `RWST` |
| `Control.Monad.Trans.Class`, `Control.Monad.IO.Class` | `lift`, `liftIO` |
| `Data.Functor.Identity`, `Control.Monad.Identity` | `Identity`, `runIdentity` |

## Usage

```haskell
import Control.Monad.State

counter :: State Int Int
counter = do
    modify (+1)
    x <- get
    put (x * 10)
    gets (+1)

main :: IO ()
main = print (runState counter 1)   -- (21,20)
```

## What works today

Checked against GHC 9.8.2 on 2026-10-20:

- `State`: `get`, `put`, `modify`, `gets`, `runState`, `evalState`
- `Reader`: `ask`, `local`, `runReader`
- `ReaderT r IO` and `StateT s IO` with `lift` / `liftIO`, `runReaderT`, `execStateT`
- `Writer` with `tell`, `fst (runWriter w)`; `WriterT w IO` with `lift`
- `ExceptT e IO` with `throwError`, `runExceptT`

## Known gaps

- `Control.Monad.Trans.Maybe` (`MaybeT`, `runMaybeT`) is not available on the default path.
- `runExcept` on the pure `Except` monad is a stub that aborts.
- A two-transformer stack over `IO`, `ReaderT Int (StateT Int IO)`, runs but reads `0` for the state instead of the value passed to `runStateT`.
- `print (snd (runWriter w))`, `print (execWriter w)` and `print` of a `runExceptT` result print a pointer. Consuming the output instead works: for a `WriterT [Int] IO`, `length w` matches GHC.
- `runState (mapM f xs) s` prints nothing.
