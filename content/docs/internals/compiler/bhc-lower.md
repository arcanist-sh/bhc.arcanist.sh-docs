+++
title = "bhc-lower"
description = "AST to HIR lowering"
weight = 40
+++

# bhc-lower

AST to HIR lowering for the Basel Haskell Compiler.

## Overview

This crate transforms the parsed AST (`bhc-ast`) into High-level IR (`bhc-hir`). It desugars surface syntax, resolves names to `DefId`s, loads imported modules, and expands the supported Template Haskell declaration splices.

```rust
let mut ctx = LowerContext::new();
let config = LowerConfig { include_builtins: true, warn_unused: false, search_paths };
let hir_module = lower_module(&mut ctx, &ast_module, &config)?;
```

`lower_module_with_cache` takes a pre-seeded `ModuleCache`, so that in multi-module builds later modules resolve imports from modules compiled earlier instead of reloading them from disk.

## Modules

| Module | Responsibility |
|--------|----------------|
| `lower` | `lower_module`, `LowerConfig`, expression/pattern/declaration lowering, pattern-synonym expansion, fixity declarations |
| `desugar` | `desugar_do`, `desugar_list_comp`, `desugar_guarded_rhs`, `desugar_guarded_rhs_or` |
| `resolve` | `resolve_var`, `resolve_type`, `resolve_constructor`, `bind_pattern`, `collect_module_definitions` |
| `context` | `LowerContext`, `Scope`, `DefKind`, `DefMap`, builtin definitions, and the classes, instances, constructors and type synonyms loaded from `.bhi` interfaces |
| `loader` (public) | `ModuleCache`, `ModuleExports`, `load_module`, `find_module_file`, `collect_exports`, `apply_import_spec`, `register_imported_names` |
| `th_expand` | `expand_th_splices`: Template Haskell declaration derivers |

## Key Transformations

### Do-Notation

```haskell
do { x <- e1; e2 }     -- e1 >>= \x -> e2
do { e1; e2 }          -- e1 >> e2
do { let x = e1; e2 }  -- let x = e1 in e2
```

### List Comprehensions

```haskell
[e | x <- xs, p, y <- ys]
-- concatMap (\x -> if p then concatMap (\y -> [e]) ys else []) xs
```

### Guards

```haskell
f x | g1 = e1 | g2 = e2 | otherwise = e3
-- if g1 then e1 else if g2 then e2 else e3

f mx | Just x <- mx = e
-- case mx of { Just x -> e; _ -> <fallback> }
```

The fallback is a pattern-match error, or the next `case` alternative when `desugar_guarded_rhs_or` is used.

### Template Haskell

There is no `Q` evaluator. `expand_th_splices` recognizes known derivers applied to a quoted type name and synthesizes the declarations from the module's own `data`/`newtype` declarations. `$(makeLenses ''T)` generates a van Laarhoven lens for each `_field` of `T`. Unrecognized declaration splices are dropped. Expression splices `$(e)` are parsed as an application of the builtin `$splice`, which is given a fresh result type at each use.

## Name Resolution

The lowering pass resolves names to their definitions:

- Local variables and pattern bindings
- Top-level bindings, types, constructors and classes
- Imported names, following import lists, `hiding`, qualified imports and re-exports
- Names from separately compiled modules, through `.bhi` interface entries
