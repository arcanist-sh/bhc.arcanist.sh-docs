+++
title = "bhc-core"
description = "Core IR for BHC"
weight = 31
+++

# bhc-core

Core Intermediate Representation for the Basel Haskell Compiler.

## Overview

Core IR is the small, explicit language that the optimizer works on and that the native (LLVM) and WASM backends consume. It is modelled on GHC Core: type classes have already been turned into dictionary arguments, pattern matching into `case`, and bindings are split into `NonRec`/`Rec` groups. The crate also contains the Core optimization passes and a Core interpreter.

## Characteristics

- **Typed by annotation**: every `Var` carries a `bhc_types::Ty` and `Expr::ty()` computes a type, but many nodes still hold `Ty::Error` placeholders; types from the type checker are threaded in for function-typed variables (see `spec/BHC-BRIEF-0002`).
- **Explicit**: type applications, casts and coercions have their own constructors.
- **Serializable**: all Core types derive `Serialize`/`Deserialize`; the driver writes each module's Core bodies to a `.bhc` sidecar next to its `.bhi` for cross-module specialization.

## Key Types

```rust
pub enum Expr {
    Var(Var, Span),
    Lit(Literal, Ty, Span),
    App(Box<Expr>, Box<Expr>, Span),
    TyApp(Box<Expr>, Ty, Span),         // f @Int
    Lam(Var, Box<Expr>, Span),
    TyLam(TyVar, Box<Expr>, Span),      // /\a -> e
    Let(Box<Bind>, Box<Expr>, Span),
    Case(Box<Expr>, Vec<Alt>, Ty, Span),
    Lazy(Box<Expr>, Span),              // H26 lazy escape hatch
    Cast(Box<Expr>, Coercion, Span),
    Tick(Tick, Box<Expr>, Span),
    Type(Ty, Span),
    Coercion(Coercion, Span),
}

pub enum Bind { NonRec(Var, Box<Expr>), Rec(Vec<(Var, Box<Expr>)>) }
pub struct Alt { pub con: AltCon, pub binders: Vec<Var>, pub rhs: Expr }
pub enum AltCon { DataCon(DataCon), Lit(Literal), Default }
```

`CoreModule` holds the module name, `bindings`, foreign imports/exports, the `OverloadedStrings` flag and `constructors` (layout metadata for codegen). `Literal` is `Int(i64)`, `Integer(i128)`, `Float`, `Double`, `Char` or `String(Symbol)`.

## Optimization Passes

The driver runs these in order (when the optimization level is not `None`):

| Pass | Module / function | When |
|------|-------------------|------|
| Monad monomorphization | `monomorphize::monomorphize_module` | Before the simplifier; specializes polymorphic-monad functions at the concrete transformer stacks they are used at, including bodies imported from `.bhc` sidecars |
| Simplifier | `simplify::simplify_module(&mut module, &SimplifyConfig)` | Beta reduction, case-of-known-constructor, case-of-case, dead-binding elimination, constant folding, inlining; iterated to a fixpoint |
| Demand analysis + worker/wrapper | `demand::analyze_module`, `worker_wrapper::apply_worker_wrapper` | Default, Server and Realtime profiles; followed by another simplifier run |
| Dictionary specialization | `specialize::specialize_dictionaries` | All profiles; followed by another simplifier run |
| List fusion | `simplify::fuse` (`map`/`map`, `sum`/`map`, `foldl'`/`map`; `fuse_fold_enum_module` for folds over `enumFromTo`) | Numeric profile only |

`escape::analyze_escape` / `check_embedded_safe` implement the escape analysis used to reject escaping allocations in the Embedded profile.

The fusion rewrites are local Core→Core rules with documented gaps (for example `sum (map f xs)` needs `f`'s type to be known); this is separate from the Tensor IR / Loop IR fusion path, which native codegen does not yet consume.

## Example

```haskell
-- Source
id True

-- Core (explicit type application)
id @Bool True
```

## Evaluator

`eval` is a tree-walking Core interpreter used by `bhci` and the playground. It supports lazy (Default profile) and strict (Numeric profile) evaluation:

```rust
use bhc_core::eval::{Env, EvalMode, Evaluator, Value};
use bhc_session::Profile;

let evaluator = Evaluator::with_profile(Profile::Numeric); // or Evaluator::new(EvalMode::Lazy)
let result: Value = evaluator.eval(&core_expr, &Env::new())?;
```

Other modules: `pretty` (Core pretty-printer) and `uarray` (`UArray<T>`, an early unboxed-array prototype with `map`, `zipWith`, `fold` and `sum`).
