+++
title = "bhc-hir-to-core"
description = "HIR to Core lowering"
weight = 41
+++

# bhc-hir-to-core

HIR to Core IR lowering for the Basel Haskell Compiler.

## Overview

This crate turns name-resolved HIR, together with the type checker's results, into a `bhc_core::CoreModule`. It makes type classes explicit as dictionaries, compiles pattern matching to decision trees, groups bindings by recursion, and generates code for derived instances.

## Modules

| Module | Responsibility |
|--------|----------------|
| `context` | `LowerContext`: lowering state, constructor table (`ConstructorInfo`), imported classes/instances |
| `expr` | Expression lowering, including method dispatch at use sites |
| `pattern` | Match-matrix pattern compilation and exhaustiveness/overlap checking |
| `binding` | Splits `let` groups into `NonRec` and `Rec` bindings |
| `dictionary` | `ClassRegistry`, `ClassInfo`, `InstanceInfo`, `DictContext`; dictionary construction, `select_method`, `select_superclass` |
| `deriving` | `DerivingContext` and `DerivedInstance` |

## Entry Points

```rust
use bhc_hir_to_core::{lower_module, lower_module_with_imports};

let core: CoreModule = lower_module(&hir_module)?;

// The driver uses the full form:
let core = lower_module_with_imports(
    &hir_module,
    Some(&defs),                 // DefMap: DefId -> DefInfo
    Some(&type_schemes),         // TypeSchemeMap: DefId -> Scheme
    Some(&imported_constructors),
    Some(&expr_types),           // ExprTypeMap: Span -> Ty (TypedModule::expr_types)
    Some(&resolved_expr_types),
    Some((&classes, &instances)), // from module interfaces
    Some(&type_aliases),
)?;
```

`lower_module_with_defs` and `lower_module_with_defs_and_constructors` are intermediate variants. Errors are `LowerError::{Internal, PatternError, Multiple}`.

## Key Transformations

### Dictionary Passing

Class constraints become explicit dictionary parameters, and method calls become selections from a dictionary. A dictionary is a tuple of method implementations, with superclass dictionaries as leading fields.

```haskell
-- Source
show :: Show a => a -> String

-- Core (conceptually; the dictionary is an extra leading argument)
show :: forall a. ShowDict a -> a -> String
```

### Instance Resolution

When the instance is known at the call site (from the constraint or from the per-span types recorded by the type checker), the dictionary is built once and bound to a variable named after the class and types, and the method is selected from it (`$sel_N`) or called directly:

```haskell
-- Source
show 42

-- Core (conceptually)
let $dictShow_Int = (...) in $sel_0 $dictShow_Int 42
```

Instances loaded from `.bhi` interfaces are registered through `lower_module_with_imports`, so calls also resolve to imported instance implementations. Class functional dependencies from imported modules are registered with `set_imported_class_fundeps`.

### Pattern Compilation

Multi-equation definitions are compiled with a column-based (Augustsson/Sestoft) match matrix into nested `case` expressions, sharing tests across equations. Guards fall through to the next row. Non-exhaustive matches produce warnings.

```haskell
-- HIR
f (Just (x:xs)) = ...
f (Just [])     = ...
f Nothing       = ...

-- Core (nested case)
f arg = case arg of
  Just tmp -> case tmp of
    (:) x xs -> ...
    []       -> ...
  Nothing -> ...
```

### Deriving

`deriving` clauses on `data` generate `Eq`, `Ord`, `Show`, `Read`, `Enum`, `Bounded`, `Functor`, `Foldable`, `Traversable` and `Generic` instances (and an empty `NFData`). Newtypes additionally support derived `Functor`, `Applicative` and `Monad` over the wrapped type.

### Newtypes

Newtype constructors are marked `is_newtype` in `ConstructorInfo` and passed to codegen, which treats them as the identity at runtime. Lowering does not emit `Cast` coercions for them.
