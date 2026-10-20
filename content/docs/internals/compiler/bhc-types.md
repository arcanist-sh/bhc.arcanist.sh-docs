+++
title = "bhc-types"
description = "Type representation for BHC"
weight = 20
+++

# bhc-types

Type representation for the Basel Haskell Compiler.

## Overview

This crate defines the representation of types shared by the type checker, HIR-to-Core lowering, Core, Tensor IR and the backends, and the `.bhi` interface format. It covers ordinary Haskell types, unboxed primitive types and type-level naturals and lists for shape-indexed tensors.

## Key Types

| Type | Description |
|------|-------------|
| `Ty` | A type |
| `Scheme` | A polymorphic type: quantified variables, constraints, body |
| `TyVar` | A type variable (`id`, `kind`) |
| `TyCon` | A type constructor (`name`, `kind`) |
| `Constraint` | A class constraint (`class`, `args`, `span`); multi-parameter via `new_multi` |
| `Subst` | Substitution from type variables to types |
| `Kind` | The kind of a type |
| `PrimTy` | Unboxed primitive types |
| `TyNat`, `TyList` | Type-level naturals and lists (modules `nat`, `ty_list`) |
| `TypeError` | `Mismatch`, `OccursCheck`, `UnboundVar`, `KindMismatch` |

## Type Representation

```rust
pub enum Ty {
    Var(TyVar),                   // a
    Con(TyCon),                   // Int, Maybe, []
    Prim(PrimTy),                 // Int#, Double#
    App(Box<Ty>, Box<Ty>),        // Maybe Int
    Fun(Box<Ty>, Box<Ty>),        // a -> b
    Tuple(Vec<Ty>),               // (a, b, c); () is Tuple(vec![])
    List(Box<Ty>),                // [a]
    Forall(Vec<TyVar>, Box<Ty>),  // forall a. a -> a
    Error,                        // placeholder for error recovery
    Nat(TyNat),                   // 1024 in Tensor '[1024] Float
    TyList(TyList),               // '[1024, 768]
}

pub struct Scheme {
    pub vars: Vec<TyVar>,
    pub constraints: Vec<Constraint>,
    pub ty: Ty,
}
```

Constructors include `Ty::unit`, `Ty::fun`, `Ty::list`, `Ty::nat_lit`, `Ty::shape`, and `Scheme::mono`, `Scheme::poly`, `Scheme::qualified`. `Subst::apply` makes one pass; `Subst::apply_resolved` applies to a fixpoint. `types_match`, `types_match_multi` and `types_equal` are used for instance matching.

## Built-in Types

This crate has no named constants for `Int`, `Bool` and so on: those are `Ty::Con` values registered by the type checker (`bhc_typeck::builtins`). The unboxed types are `PrimTy`:

| `PrimTy` | Haskell type |
|----------|--------------|
| `I32`, `I64` | `Int32#`, `Int64#`/`Int#` |
| `U32`, `U64` | `Word32#`, `Word64#`/`Word#` |
| `F32`, `F64` | `Float#`, `Double#` |
| `Char` | `Char#` |
| `Addr` | `Addr#` |

The `dyn_tensor` module builds the types for dynamically shaped tensors (`DynTensor`, shape witnesses).

## Kind System

```rust
pub enum Kind {
    Star,                        // *
    Arrow(Box<Kind>, Box<Kind>), // k1 -> k2
    Constraint,                  // Constraint
    Var(u32),                    // kind variable, for inference
    Nat,                         // type-level naturals
    List(Box<Kind>),             // [k], e.g. [Nat] for shapes
}
```

`Kind::tensor_kind()` is `[Nat] -> * -> *`.

## IO Model Switch

`io_closure_mode()` reads `BHC_IO_MODEL=closure`. In that mode IO actions run only when executed, and the monad transformers come from the `bhc-transformers` Haskell library rather than compiler builtins; `TRANSFORMER_LIBRARY_NAMES` lists the names that must then not be treated as builtins. Every compiler stage reads this one switch.
