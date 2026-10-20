+++
title = "bhc-ast"
description = "Abstract syntax tree for bhc"
weight = 11
+++

# bhc-ast

Abstract syntax tree definitions for the Basel Haskell Compiler.

## Overview

This crate (a single `lib.rs`) defines the tree produced by `bhc-parser`. It keeps the surface syntax close to the source (parentheses, infix applications, qualified names, record syntax) so that diagnostics can point at what the user wrote. Nodes are ordinary owned values (`Box`, `Vec`); names are `bhc_intern::Ident`s.

## Key Types

| Type | Description |
|------|-------------|
| `Module` | Doc comment, pragmas, optional `ModuleName`, export list, imports, declarations, span |
| `ImportDecl`, `ImportSpec`, `Export` | Import and export lists |
| `Pragma`, `PragmaKind`, `Extension` | Pragmas, including `LANGUAGE` extensions |
| `Decl` | Top-level and local declarations |
| `Expr` | Expressions |
| `Pat` | Patterns |
| `Type` | Type syntax |
| `ExprId`, `PatId`, `TypeId`, `DeclId` | Index types declared with `bhc_index::define_index!` |

`Decl` covers type signatures, function bindings (`FunBind` with `Clause`s and guarded `Rhs`), `data`/`newtype`/`type`, GADT constructors, classes with associated types and functional dependencies, instances, `foreign` declarations, fixity, standalone deriving, pattern synonyms, type and data families and their instances, and `Splice` for top-level Template Haskell declaration splices.

## Expression Types

Every variant carries its `Span` as the last field:

```rust
pub enum Expr {
    Var(Ident, Span),                       // x
    QualVar(ModuleName, Ident, Span),       // M.x
    Con(Ident, Span),                       // Just
    QualCon(ModuleName, Ident, Span),
    Lit(Lit, Span),                         // 42, 'c', "hello"
    App(Box<Expr>, Box<Expr>, Span),        // f x
    Lam(Vec<Pat>, Box<Expr>, Span),         // \x -> e
    Let(Vec<Decl>, Box<Expr>, Span),
    If(Box<Expr>, Box<Expr>, Box<Expr>, Span),
    Case(Box<Expr>, Vec<Alt>, Span),
    Do(Vec<Stmt>, Span),
    Tuple(Vec<Expr>, Span),
    List(Vec<Expr>, Span),
    ArithSeq(ArithSeq, Span),               // [a..b]
    ListComp(Box<Expr>, Vec<Stmt>, Span),
    RecordCon(Ident, Vec<FieldBind>, bool, Span),
    RecordUpd(Box<Expr>, Vec<FieldBind>, Span),
    Infix(Box<Expr>, Ident, Box<Expr>, Span),
    Neg(Box<Expr>, Span),
    Paren(Box<Expr>, Span),
    Ann(Box<Expr>, Type, Span),             // e :: T
    TypeApp(Box<Expr>, Type, Span),         // f @Int
    Lazy(Box<Expr>, Span),                  // H26 lazy { ... }
    Wildcard(Span),
    // ... plus QualRecordCon
}
```

## Pattern Types

```rust
pub enum Pat {
    Wildcard(Span),                         // _
    Var(Ident, Span),                       // x
    Lit(Lit, Span),                         // 42
    Con(Ident, Vec<Pat>, Span),             // Just x
    Infix(Box<Pat>, Ident, Box<Pat>, Span), // x : xs
    Tuple(Vec<Pat>, Span),
    List(Vec<Pat>, Span),
    Record(Ident, Vec<FieldPat>, bool, Span),
    As(Ident, Box<Pat>, Span),              // x@(Just y)
    Lazy(Box<Pat>, Span),                   // ~p
    Bang(Box<Pat>, Span),                   // !p
    View(Box<Expr>, Box<Pat>, Span),        // (f -> p)
    // ... plus QualCon, QualRecord, Paren, Ann
}
```

`Type` has variables, constructors, application, functions, tuples, lists, `forall`, constrained types, promoted lists and type-level naturals, strictness/laziness annotations and infix type operators.

## Span Preservation

Spans are stored inline in each variant rather than in a wrapper struct. `Expr::span()`, `Pat::span()` and `Type::span()` return them.
