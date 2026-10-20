+++
title = "bhc-hir"
description = "High-level IR for bhc"
weight = 30
+++

# bhc-hir

High-level Intermediate Representation for the Basel Haskell Compiler.

## Overview

HIR is produced by `bhc-lower` from the AST. It is desugared and name-resolved: every variable and constructor reference is a `DefRef` pointing at a `DefId`, while patterns, guards and type signatures are kept so that the type checker and pattern compiler can work on them. The crate is a single `lib.rs` of data definitions; all HIR types derive `Serialize`/`Deserialize`.

## Pipeline Position

```
AST  →  [bhc-lower]  →  HIR  →  [bhc-typeck]  →  HIR + types  →  [bhc-hir-to-core]  →  Core
```

The type checker does not rewrite HIR; it produces a separate `TypedModule` (inferred schemes per `DefId` and types keyed by expression span) that `bhc-hir-to-core` reads alongside the HIR.

## Desugaring Done Before HIR

| From | To | Where |
|------|-----|-------|
| Do-notation | `>>=` / `>>` chains | `bhc-lower` `desugar.rs` |
| List comprehensions | `concatMap` | `bhc-lower` `desugar.rs` |
| `where` clauses | `Let` around the body | `bhc-lower` |
| Operator sections | Lambdas | `bhc-parser` |
| TH declaration splices (`$(makeLenses ''T)` and other recognized derivers) | Synthesized declarations, no `Q` evaluation | `bhc-lower` `th_expand.rs` |

Guards are not removed: `Equation` and `CaseAlt` carry `guards: Vec<Guard>`, and are compiled to decision trees later in `bhc-hir-to-core`.

## Key Types

| Type | Description |
|------|-------------|
| `Module` | Name, exports, imports, `items: Vec<Item>` |
| `Item` | `Value`, `Data`, `Newtype`, `TypeAlias`, `Class`, `Instance`, `Fixity`, `Foreign`, `StandaloneDeriving`, `PatternSynonym`, type/data families and instances |
| `ValueDef` / `Equation` | A binding with optional `Scheme` and one equation per clause |
| `DefId`, `DefRef` | Definition identifier; reference with span |
| `HirId` | Node identifier (implements `bhc_index::Idx`) |

```rust
pub enum Expr {
    Lit(Lit, Span),
    Var(DefRef),
    Con(DefRef),
    App(Box<Expr>, Box<Expr>, Span),
    Lam(Vec<Pat>, Box<Expr>, Span),
    Let(Vec<Binding>, Box<Expr>, Span),
    Case(Box<Expr>, Vec<CaseAlt>, Span),
    If(Box<Expr>, Box<Expr>, Box<Expr>, Span),
    Tuple(Vec<Expr>, Span),
    List(Vec<Expr>, Span),
    Record(DefRef, Vec<FieldExpr>, Span),
    FieldAccess(Box<Expr>, Symbol, Span),
    RecordUpdate(Box<Expr>, Vec<FieldExpr>, Span),
    Ann(Box<Expr>, Ty, Span),          // Ty from bhc-types
    TypeApp(Box<Expr>, Ty, Span),
    Error(Span),                       // error recovery
}
```

`Pat` has `Wild`, `Var(Symbol, DefId, Span)`, `Lit`, `Con`, `RecordCon`, `As`, `Or`, `Ann`, `View` and `Error`; `Pat::bound_vars()` lists the variables a pattern binds.

## Example

```haskell
-- Source
do
  x <- getLine
  print x

-- HIR
getLine >>= (\x -> print x)
```
