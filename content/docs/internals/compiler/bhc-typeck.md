+++
title = "bhc-typeck"
description = "Type checking for BHC"
weight = 21
+++

# bhc-typeck

Type inference and checking for the Basel Haskell Compiler.

## Overview

This crate type checks HIR modules produced by [bhc-lower](@/docs/internals/compiler/bhc-lower.md). It implements Hindley-Milner inference (Algorithm W) with let-polymorphism, SCC-based binding groups, signature checking and error recovery, plus type classes with superclasses and functional dependencies, type families, GADTs, kind checking and type-level naturals for tensor shapes.

## Key Types

| Type | Description |
|------|-------------|
| `TyCtxt` | The inference context: substitution, constraints, environment, diagnostics |
| `TypeEnv` | Scoped type environment (persistent maps from `im`) |
| `DataConInfo` | Constructor information used by pattern checking |
| `KindEnv` | Kind environment for `kind_check` |
| `TypedModule` | The result: HIR plus types (below) |
| `ImportedClasses` | Classes, fundeps, instances and record field types loaded from `.bhi` interfaces |

```rust
pub struct TypedModule {
    pub hir: Module,
    pub expr_types: FxHashMap<Span, Ty>,
    pub resolved_expr_types: FxHashMap<Span, Ty>,
    pub def_schemes: FxHashMap<DefId, Scheme>,
    pub type_aliases: FxHashMap<Symbol, (Vec<TyVar>, Ty)>,
}
```

Each expression's inferred type is recorded under its source `Span`. HIR-to-Core lowering looks types up by span to choose which instance or monad a use site refers to. `resolved_expr_types` applies the final substitution to a fixpoint and serves only those dispatch decisions. `type_aliases` lets interface generation expand synonyms in exported signatures.

## Usage

```rust
use bhc_typeck::{type_check_module, type_check_module_with_imports, ImportedClasses};

// Single module
let typed = type_check_module(&hir_module, file_id)?;

// With definitions, imported synonyms and imported classes/instances
let typed = type_check_module_with_imports(
    &hir_module, file_id, Some(&def_map), &imported_aliases,
    &ImportedClasses { classes, fundeps, instances, record_field_types },
)?;
```

Errors are returned as `Vec<Diagnostic>`.

## Type Inference Algorithm

1. **Binding group analysis**: SCCs of top-level bindings (`petgraph`)
2. **Constraint generation**: Walk HIR, generate equalities and class constraints
3. **Unification**: Solve equalities by substitution, reducing type families
4. **Constraint solving**: Superclass propagation, fundep improvement, instance lookup, numeric defaulting
5. **Generalization**: Quantify free variables at let-bindings

## Type Classes

Type classes are compiled by dictionary passing, including constraints on individual class methods (`query :: Monoid c => ...`). The type checker infers and checks constraints; building and passing the dictionaries happens in `bhc-hir-to-core`, using the span-keyed types above:

```haskell
show 42
-- after HIR-to-Core: show applied to the Show Int dictionary, then 42
```

## Modules

`builtins` (primitive types, constructors and Prelude signatures), `kind_check`, `type_families`, `nat_solver` (type-level naturals), `shape_bridge` (type-level shapes to Tensor IR shapes), `shape_diagrams` (ASCII diagrams for shape errors), `suggest` ("did you mean" hints).

## Error Messages

Diagnostics carry codes such as `E0001` (type mismatch) and render through `bhc-diagnostics`:

```
error[E0001]: type mismatch: expected `Int`, found `String`
  --> src/Main.hs:10:9
   |
10 |     x + "hello"
   |         ^^^^^^^ expected `Int`
```
