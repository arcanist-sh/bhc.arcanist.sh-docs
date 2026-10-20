+++
title = "Compiler Architecture"
description = "How BHC is put together: the pipeline from source to binary, the intermediate representations, the optimizer, the backends and the runtime."
weight = 0
+++
BHC is a multi-stage optimizing compiler written in Rust. This page documents the internal architecture, intermediate representations, and compilation pipeline as they exist in the source today, and marks the places where a part of the design is implemented but not yet on the compiled-code path.

> Most of the pipeline below runs on every build. Three parts are built and tested but not yet wired into compiled programs: the garbage collector (compiled programs allocate without freeing), the numeric profile's Tensor IR and Loop IR fusion (native code generation compiles Core directly), and the work-stealing scheduler (no compiled Haskell code reaches it). Each is marked where it appears. The [status page](https://arcanist.sh/bhc/status/) tracks what is wired today. The crate pages in this section document each part in detail.

## High-Level Pipeline

A module goes through the same frontend and Core optimizer whatever the profile or target. The driver (`bhc-driver`) parses it, lowers the AST to HIR (resolving names and desugaring), type-checks the HIR, lowers it to Core IR, and optimizes Core. Code generation then branches on the target, not on an IR per profile: native builds go Core → LLVM IR via `bhc-codegen`, WebAssembly builds go Core → WASM via `bhc-wasm`. The numeric profile additionally lowers Core to Tensor IR and Loop IR, whose kernels feed the kernel report, the WASM module and the GPU backends.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BHC COMPILATION PIPELINE                            │
└─────────────────────────────────────────────────────────────────────────────┘

  Source Files (.hs)        (CPP preprocessing when the module asks for it)
        │
        ▼
┌───────────────────┐
│  LEX + PARSE      │  Layout rule, recursive descent, fixities → AST
└───────────────────┘
        │
        ▼
┌───────────────────┐
│  LOWER (AST→HIR)  │  Name resolution, desugaring, imports from .bhi
└───────────────────┘
        │
        ▼
┌───────────────────┐
│  TYPE CHECK       │  Hindley-Milner + classes; span-keyed types out
└───────────────────┘
        │
        ▼
┌───────────────────┐
│  HIR → CORE       │  Pattern compilation, dictionary passing
└───────────────────┘
        │
        ▼
┌───────────────────┐
│  CORE OPTIMIZER   │  Monomorphize, simplify, demand + W/W, specialize
└───────────────────┘
        │
        ├──────────────────────────┬─────────────────────────────┐
        ▼                          ▼                             ▼
┌───────────────┐          ┌───────────────┐          ┌──────────────────────┐
│ LLVM CODEGEN  │          │ WASM CODEGEN  │          │ TENSOR IR → LOOP IR  │
│ Core → LLVM   │          │ Core → WASM   │          │ (numeric profile)    │
│ (native)      │          │ (WASI)        │          │ fusion, vectorize    │
└───────────────┘          └───────────────┘          └──────────────────────┘
        │                          │                    │          │
        ▼                          │                    ▼          ▼
┌───────────────┐                  │             kernels in   PTX / Metal
│    LINKER     │                  │             WASM module  kernels
│ cc + RTS .a   │                  │             (and kernel report)
└───────────────┘                  │
        │                          ▼
        ▼                     module.wasm
    Executable
```

There is no separate STG or "direct" IR. The default and edge profiles share the Core → backend path; what the profile changes is which optimizer passes run, the RTS configuration, and (for numeric) the extra Tensor/Loop IR stage.

## Frontend Architecture

The frontend turns source text into type-checked HIR. Each phase is its own crate: `bhc-lexer`, `bhc-parser` (producing `bhc-ast`), `bhc-lower` (producing `bhc-hir`), and `bhc-typeck`.

### Lexer and Parser

The lexer converts source text into a token stream. Haskell's layout rule is handled there: the lexer keeps a stack of indentation contexts and inserts virtual braces and semicolons, so the parser never has to reason about indentation.

The parser is a hand-written recursive descent parser. Infix expressions are parsed by precedence climbing, so operator fixity is resolved during parsing: before parsing a module, the parser scans its tokens for `infixl`/`infixr`/`infix` declarations, and those take priority over a built-in table that covers the Prelude, `base` and other common operators. Fixity declarations do not yet travel across modules through interface files, so an operator declared in one module and used in another gets its fixity from the built-in table (or the `infixl 9` default). The parser builds the AST directly; there is no separate concrete syntax tree. Parse errors are reported as diagnostics with source spans.

When a module enables `{-# LANGUAGE CPP #-}` (or `-XCPP` is passed), the driver runs it through BHC's own C preprocessor first, honouring `-D` defines.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND DETAIL                                │
└─────────────────────────────────────────────────────────────────────────────┘

   Source Text  (after optional CPP)
        │
        ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                               LEXER                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   Layout    │  │  Keywords   │  │  Operators  │  │  Literals   │      │
│  │   Stack     │──│  & Idents   │──│  & Symbols  │──│  & Strings  │      │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘      │
│         │                                                                  │
│         ▼                                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐      │
│  │  Token Stream with Layout-Inserted Braces and Semicolons        │      │
│  └─────────────────────────────────────────────────────────────────┘      │
└───────────────────────────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                               PARSER                                       │
│                                                                            │
│   ┌─────────────────────────────────────────────────────────────────┐     │
│   │                    Recursive Descent Parser                      │     │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │     │
│   │  │  Module  │  │  Decls   │  │  Exprs   │  │ Patterns │        │     │
│   │  │  Header  │──│  Parser  │──│ (prec.   │──│  Parser  │        │     │
│   │  │          │  │          │  │ climbing)│  │          │        │     │
│   │  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │     │
│   └─────────────────────────────────────────────────────────────────┘     │
│         ▲                                                                  │
│         │  fixities: module's own declarations, then built-in table        │
└───────────────────────────────────────────────────────────────────────────┘
        │
        ▼
   Abstract Syntax Tree (AST)
```

### Name Resolution and Desugaring

`bhc-lower` lowers the AST to HIR in a single pass that does name resolution and desugaring together. As it descends into the AST it builds lexical scopes and assigns every definition a `DefId`; every use of a name in the HIR refers to its definition by that ID, so later phases never deal with scoping rules.

Imports are resolved in the same pass. The loader finds an imported module either as source on the search paths (`-I`, `--package-dir`, the stdlib path) or as a compiled `.bhi` interface in `--hidir` or a `--package-db`, then binds the names it exports, handling qualified imports, aliases, `hiding` lists and re-exports. Desugaring also happens here: do-notation becomes `>>=` chains, list comprehensions become `concatMap`, guards become case expressions, and `where` clauses become `let`s. Template Haskell declaration splices for a handful of common derivers (such as `makeLenses`) are expanded by native synthesis in this pass.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     NAME RESOLUTION AND DESUGARING                          │
└─────────────────────────────────────────────────────────────────────────────┘

                              AST (names are strings)
                                       │
          ┌────────────────────────────┼────────────────────────────┐
          ▼                            ▼                            ▼
   ┌─────────────┐              ┌─────────────┐              ┌─────────────┐
   │   Import    │              │   Scope     │              │  Desugar    │
   │   Loader    │              │  Tracking   │              │             │
   │             │              │             │              │ - do →      │
   │ - Source on │              │ - Lexical   │              │   >>=       │
   │   search    │              │   scopes    │              │ - list      │
   │   paths     │              │ - DefIds    │              │   comps →   │
   │ - .bhi in   │              │   for every │              │   concatMap │
   │   hidir /   │              │   binder    │              │ - guards →  │
   │   package   │              │ - Qualified │              │   case      │
   │   DB        │              │   names     │              │             │
   └─────────────┘              └─────────────┘              └─────────────┘
          │                            │                            │
          └────────────────────────────┼────────────────────────────┘
                                       ▼
                           ┌───────────────────┐
                           │       HIR         │
                           │  (every name has  │
                           │   a DefId)        │
                           └───────────────────┘
```

### Type Inference

`bhc-typeck` implements Hindley-Milner inference (Algorithm W) over HIR. Bindings are first split into strongly connected components so mutually recursive groups are inferred together; each group is inferred, unified, and generalized at its `let`. User-written signatures are checked against the inferred types, and inference continues past errors using an error type, so one mistake does not hide the rest.

**Unification** applies the current substitution to both sides, tries to reduce type family applications, binds a type variable after an occurs check, and recurses structurally on constructors and function types; anything else is a type mismatch. Beyond plain HM, the checker has kind checking, open and closed type families, functional dependencies, and a small solver for type-level naturals used by tensor shapes.

**Class constraints** are collected during inference and solved at the end: superclasses are propagated (`Ord a` implies `Eq a`), functional dependencies are applied, each constraint is matched against the known instances, ambiguous numeric variables are defaulted (`Fractional` wins over `Num`, so such a variable defaults to `Double`), and an unsatisfiable constraint on a concrete type is reported as a "no instance" error. The typechecker does not build dictionaries itself; that happens when HIR is lowered to Core.

The output is a `TypedModule`: a type scheme per definition plus the inferred type of every expression, keyed by source span (`expr_types`, and a fully resolved variant, `resolved_expr_types`). HIR → Core lowering reads these to pick which instance a use site means, and the Core optimizer reads them for monomorphization.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            TYPE INFERENCE                                   │
└─────────────────────────────────────────────────────────────────────────────┘

                                HIR
                                 │
                                 ▼
                     Binding groups (SCC order)
                                 │
                                 ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                     INFERENCE (Algorithm W)                                │
│                                                                            │
│    Expression          Rule                                                │
│    ──────────          ────                                                │
│    x                   instantiate scheme of x                             │
│    \x -> e             fresh α for x, infer e : β, result α → β            │
│    f x                 infer f, infer x : α, unify f with α → β, result β  │
│    let x = e1 in e2    infer e1, generalize, infer e2 with x bound         │
│    method use          instantiate, emit class constraint (C τ)            │
│                                                                            │
└───────────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                         SOLVING                                            │
│                                                                            │
│   ┌────────────────────────────────────────────────────────────────┐      │
│   │                    Unification (substitution)                   │      │
│   │                                                                 │      │
│   │   τ₁ ~ τ₂                                                       │      │
│   │    │                                                            │      │
│   │    ├── Type family application? ─────► Reduce, then retry      │      │
│   │    │                                                            │      │
│   │    ├── One side a type var? ─────────► Occurs check, then bind  │      │
│   │    │                                                            │      │
│   │    ├── Both constructors? ───────────► Recurse on arguments     │      │
│   │    │                                                            │      │
│   │    └── Mismatch? ────────────────────► Type error (E0001)       │      │
│   │                                                                 │      │
│   └────────────────────────────────────────────────────────────────┘      │
│                                                                            │
│   ┌────────────────────────────────────────────────────────────────┐      │
│   │                   Class Constraints                             │      │
│   │                                                                 │      │
│   │   For constraint (C τ):                                         │      │
│   │    1. Propagate superclasses, apply fundeps                     │      │
│   │    2. Find an instance matching τ (and its context)             │      │
│   │    3. Type variable left over?  default it                      │      │
│   │    4. Concrete type, no instance?  report an error              │      │
│   │                                                                 │      │
│   └────────────────────────────────────────────────────────────────┘      │
└───────────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
          TypedModule: schemes + span-keyed expression types
```

## Core IR

Core is BHC's primary intermediate representation: a small, typed functional language defined in `bhc-core`. There are only a handful of expression forms, which is what makes the optimizer tractable.

### Core Language

Core is a System F-style lambda calculus with let-bindings, case expressions and coercions. Every variable carries a type (`Var { name, id, ty }`), literals carry their type, and `Expr::ty()` computes a node's type from its structure.

Types are carried, but not yet fully populated: function-typed variables get their types from typeck's span-keyed map, while many scalar leaves still hold an error type that codegen fills in by inference. Closing that gap is ongoing work (the typed-Core effort); passes that need a type check for one and back off when it is missing.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CORE IR GRAMMAR                                │
└─────────────────────────────────────────────────────────────────────────────┘

  Expression e ::=                  (bhc_core::Expr)
      │
      ├── x                         Var       — variable (constructors too)
      │
      ├── lit                       Lit       — Int, Integer, Float, Double,
      │                                         Char, String
      ├── e₁ e₂                     App       — application
      │
      ├── e @τ                      TyApp     — type application
      │
      ├── λx:τ. e                   Lam       — lambda (type on the Var)
      │
      ├── Λα. e                     TyLam     — type abstraction
      │
      ├── let bind in e             Let       — NonRec or Rec bindings
      │
      ├── case e of alts : τ        Case      — with result type
      │
      ├── lazy { e }                Lazy      — explicit thunk in strict code
      │
      ├── e ▷ co                    Cast      — newtype / axiom coercions
      │
      ├── tick e                    Tick      — profiling / source notes
      │
      └── τ, co                     Type, Coercion — type/coercion arguments


  Bindings bind ::= NonRec x = e  |  Rec [x₁ = e₁, ..., xₙ = eₙ]


  Alternatives alt ::=              (AltCon)
      │
      ├── K x₁ ... xₙ → e           DataCon
      │
      ├── lit → e                   Lit
      │
      └── _ → e                     Default
```

### Lowering HIR to Core

`bhc-hir-to-core` translates typed HIR into Core. Most surface sugar is already gone by this point (do-notation and comprehensions were desugared in `bhc-lower`); this pass does the type-directed parts.

**Pattern matching** is compiled with the Augustsson/Sestoft column-based decision-tree algorithm. A multi-equation definition becomes a match matrix; the compiler picks the column with the most constructor information, groups rows by head constructor, emits one `case` per group, and recurses. Common tests are shared across equations, and the tree is checked for non-exhaustive matches and redundant equations, which are reported as warnings.

**Type classes** are dictionary-passed. A class's dictionary is a tuple of its superclass dictionaries followed by its methods; a method use becomes a field selection (`$sel_N dict`), an instance becomes a tuple value, and a constrained function takes its dictionaries as leading arguments. Method-level constraints (a method whose own type has a `Monoid c =>`) are dictionary-passed as well. The span-keyed types from typeck decide which instance a use site needs, including for user-defined monads and transformer stacks. Derived instances (`Eq`, `Ord`, `Show` and others) are generated here too.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          LOWERING TO CORE                                   │
└─────────────────────────────────────────────────────────────────────────────┘

  Source Haskell                      After lowering
  ──────────────                      ──────────────

  do { x <- m; e }          ═══►      m >>= \x -> e          (in bhc-lower)

  [x | x <- xs, p x]        ═══►      concatMap (\x -> if p x
                                                       then [x]
                                                       else []) xs

  f 0 y = a                 ═══►      f = \x -> \y ->
  f n y = b                             case x of
                                          0 -> a
                                          _ -> b   (n bound to x)

  class Eq a where          ═══►      dictionary = tuple of methods
    (==) :: a -> a -> Bool              (superclass dicts first)
    (/=) :: a -> a -> Bool            (==) at a use site = $sel_0 dEq

  instance Eq Int where     ═══►      $dEq_Int = (eqInt, neInt)
    ...

  elem :: Eq a => ...       ═══►      elem = \dEq -> \x -> \xs -> ...
```

## Optimization Pipeline

`bhc-core` optimizes Core with a fixed sequence of passes, run by the driver after lowering. `-O0` skips the simplifier entirely; `-O1`, `-O2` (the default) and `-O3` set its iteration limit and inlining threshold.

### Core-to-Core Transformations

In order, the passes are:

**Monad monomorphization** runs first, on freshly lowered Core. A function such as `poly :: Monad m => ... -> StateT Int m Int` is lowered once from its declared type, but codegen chooses a transformer's runtime representation from the type it sees, so a use at a concrete stack like `ExceptT e (StateT s IO)` needs its own copy. The pass reads typeck's resolved expression types, and for every top-level binding used at a concrete transformer stack it emits a specialized copy with the monad variable substituted. It works across modules: each module writes its lowered Core bodies and newtype definitions to a `.bhc` sidecar next to its `.bhi`, and an importing module loads them to specialize imported functions. Modules with no concrete monad in sight skip the pass.

**The simplifier** is the workhorse. It iterates to a fixed point (10 iterations at `-O2`): beta reduction, case-of-known-constructor and case-of-literal, case-of-case within a size budget, constant folding, inlining of single-use bindings and small multi-use ones, and elimination of dead bindings. An occurrence analysis feeds the inliner and the dead-code pass.

**Demand analysis and worker/wrapper** run for the lazy profiles (default, server, realtime). Demand analysis is a backward, per-argument strict/lazy analysis in the style of HBC's strictness analyser. Worker/wrapper then wraps the body of a function in `case` expressions that force its strict arguments on entry. BHC represents `Int` and `Int#` the same way at the LLVM level, so it does not split a separate unboxed worker the way GHC does; the forcing `case` is the whole transformation. A short simplifier run cleans up afterwards.

**Dictionary specialization** runs for all profiles. Where a method selection `$sel_N d` is applied to a dictionary that is a known tuple, it is replaced by the method itself, removing both the selection and, once the simplifier has run again, the dictionary allocation.

**Numeric list fusion** runs only in the numeric profile; see the next section.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        OPTIMIZATION PIPELINE                                │
└─────────────────────────────────────────────────────────────────────────────┘

                              Core IR (from HIR → Core)
                                       │
    ┌──────────────────────────────────┼──────────────────────────────────┐
    │                                  ▼                                   │
    │  ┌────────────────────────────────────────────────────────────────┐  │
    │  │               MONAD MONOMORPHIZATION                           │  │
    │  │                                                                │  │
    │  │   poly :: Monad m => StateT Int m Int                          │  │
    │  │   used at m = ExceptT e (StateT s IO)  ═══►  specialized copy  │  │
    │  │   (imported bodies come from .bhc sidecars)                    │  │
    │  └────────────────────────────────────────────────────────────────┘  │
    │                                  │                                   │
    │                                  ▼                                   │
    │  ┌───────────────────────────────────────────────────────────────┐  │
    │  │                        SIMPLIFIER                              │  │
    │  │                                                                │  │
    │  │   Iterates to a fixed point (O1: 4, O2: 10, O3: 15 rounds)     │  │
    │  │                                                                │  │
    │  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │  │
    │  │   │   Inlining   │  │    Beta      │  │    Dead      │        │  │
    │  │   │              │──│  Reduction   │──│   Binding    │        │  │
    │  │   │  - Once-used │  │              │  │  Elimination │        │  │
    │  │   │  - Small     │  │  (λx.e) v    │  │              │        │  │
    │  │   │   (size ≤ N) │  │    ═► e[v/x] │  │  Remove      │        │  │
    │  │   │              │  │              │  │  unused      │        │  │
    │  │   └──────────────┘  └──────────────┘  └──────────────┘        │  │
    │  │          │                  │                  │               │  │
    │  │          ▼                  ▼                  ▼               │  │
    │  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │  │
    │  │   │    Case      │  │   Constant   │  │   Case of    │        │  │
    │  │   │  of Known    │──│   Folding    │──│    Case      │        │  │
    │  │   │ Constructor  │  │              │  │              │        │  │
    │  │   │              │  │  2 + 3 ═► 5  │  │  push outer  │        │  │
    │  │   │ case K x of  │  │  not True    │  │  case into   │        │  │
    │  │   │   K y -> e   │  │    ═► False  │  │  inner alts  │        │  │
    │  │   │   ═► e[x/y]  │  │              │  │  (budgeted)  │        │  │
    │  │   └──────────────┘  └──────────────┘  └──────────────┘        │  │
    │  │                                                                │  │
    │  └────────────────────────────────────────────────────────────────┘  │
    │                                  │                                   │
    │                                  ▼                                   │
    │  ┌────────────────────────────────────────────────────────────────┐  │
    │  │          DEMAND ANALYSIS + WORKER/WRAPPER                       │  │
    │  │          (default, server, realtime profiles)                   │  │
    │  │                                                                 │  │
    │  │   f x y = case x of           f is strict in x (always eval'd) │  │
    │  │             0 -> y            f is lazy in y (not always)      │  │
    │  │             _ -> x + y                                          │  │
    │  │                                                                 │  │
    │  │   After W/W:                                                    │  │
    │  │   f = \x y -> case x of x' -> body[x := x']                     │  │
    │  │   (x forced on entry; no separate unboxed worker)               │  │
    │  │                                                                 │  │
    │  │   then: simplifier (4 rounds)                                   │  │
    │  └────────────────────────────────────────────────────────────────┘  │
    │                                  │                                   │
    │                                  ▼                                   │
    │  ┌────────────────────────────────────────────────────────────────┐  │
    │  │               DICTIONARY SPECIALIZATION                         │  │
    │  │                                                                 │  │
    │  │   let $dEq_Int = (eqInt, neInt)                                 │  │
    │  │   in ... $sel_0 $dEq_Int ...   ═══►   ... eqInt ...             │  │
    │  │                                                                 │  │
    │  │   then: simplifier (4 rounds) removes the dead dictionary       │  │
    │  └────────────────────────────────────────────────────────────────┘  │
    │                                  │                                   │
    │                                  ▼                                   │
    │  ┌────────────────────────────────────────────────────────────────┐  │
    │  │          FOLD/ENUMFROMTO FUSION  (numeric profile only)         │  │
    │  └────────────────────────────────────────────────────────────────┘  │
    │                                  │                                   │
    └──────────────────────────────────┼──────────────────────────────────┘
                                       │
                                       ▼
                              Optimized Core IR
```

Two more analyses live in `bhc-core`. Escape analysis is a backward dataflow pass that finds allocations escaping their scope; the driver runs it for the embedded profile, which has no collector and rejects programs whose allocations escape. And `bhc-core` has a Core evaluator used by tests and tooling.

### Fusion

Fusion removes intermediate data structures from pipelines such as `sum (map f xs)`. BHC does this with direct Core-to-Core rewrites rather than GHC-style stream fusion; there is no `Stream`/`Step` representation in the compiler. In the numeric profile, the simplifier enables these rewrites (`fuse_lists`):

- `map f (map g xs)` → `map (\v -> f (g v)) xs`, one traversal; the composed lambda is beta-reduced on the next round.
- `sum (map f xs)` → `foldl' (\acc x -> acc + f x) 0 xs`. This one needs to know `f` returns `Int`, so it fires for named functions with known types (`sum (map dbl xs)`) and not yet for lambdas, whose types are not yet populated in Core.
- `foldl' op z (map f xs)` → `foldl' (\acc x -> op acc (f x)) z xs`, for any `f`.

After the cleanup passes, a final numeric-only pass rewrites `sum`, `product` and `foldl'` over `enumFromTo` (optionally through an `Int -> Int` `map`) into a hoisted counting loop, so `sum [1..n]` allocates no list.

Separately, the numeric profile (or `--tensor-fusion`) lowers Core to Tensor IR, where the fusion pass in `bhc-tensor-ir` implements four guaranteed patterns as fused kernels: `map f (map g x)`, `zipWith f (map g a) (map h b)`, `sum (map f x)` and `foldl' op z (map f x)`. Those kernels are lowered to Loop IR and summarized by `--kernel-report`.

> **Status:** the Core rewrites above run in native builds and produce correct results. The Tensor IR and Loop IR fusion passes pass their tests, but native code generation compiles Core to LLVM and does not consume the fused Loop IR kernels, and native code stores every value in a pointer-sized slot. A numeric-profile native build therefore runs at the same speed as the default profile today. "Guaranteed fusion" is the standard the numeric profile is built to meet, not a description of today's native binaries. See the [status page](https://arcanist.sh/bhc/status/).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                             FUSION                                          │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │              CORE REWRITES  (numeric profile, in the simplifier)         │
  │                                                                          │
  │   map f (map g xs)         ═══►  map (\v -> f (g v)) xs                 │
  │   sum (map f xs)           ═══►  foldl' (\acc x -> acc + f x) 0 xs      │
  │                                  (f : _ -> Int known)                    │
  │   foldl' op z (map f xs)   ═══►  foldl' (\acc x -> op acc (f x)) z xs   │
  │                                                                          │
  │   sum / product / foldl' over [a..b]                                     │
  │                            ═══►  hoisted counting loop, no list          │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │              TENSOR IR KERNELS  (numeric profile, bhc-tensor-ir)         │
  │                                                                          │
  │   1. map f (map g x)                  ═══►  one kernel, composed fn     │
  │   2. zipWith f (map g a) (map h b)    ═══►  one kernel                  │
  │   3. sum (map f x)                    ═══►  one reduce-of-map kernel    │
  │   4. foldl' op z (map f x)            ═══►  one kernel                  │
  │                                                                          │
  │   Kernels → Loop IR → vectorization + parallelization analysis           │
  │   → --kernel-report;  WASM and GPU backends consume them,               │
  │     native LLVM codegen does not yet                                     │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘
```

## Backend Architecture

After optimization, the driver hands Core to a backend chosen by the target: LLVM for native targets, `bhc-wasm` for `wasm32-wasi`, or the GPU path for `--emit ptx` and GPU targets.

### Native Code Generation and Evaluation

Native code generation (`bhc-codegen`) lowers Core directly to LLVM IR through `inkwell`. There is no STG stage or STG machine: no info tables, no `R1`/`Sp`/`Hp` register model, and no closure entry code. Instead, Core functions become LLVM functions and values are uniform pointer-sized words, with a few simple heap layouts:

- **Data constructors** are `{ i64 tag, field₀, field₁, ... }`, with each field one pointer-sized slot.
- **Closures** are `{ fn_ptr, i64 env_size, env[env_size] }`. Every compiled function takes an environment pointer as its first parameter; unsaturated calls and lambdas capture their free variables in a closure, and calling a closure passes its environment.
- **Thunks** are `{ i64 tag, eval_fn, i64 env_size, env[env_size] }`. The tag is `-1` while unevaluated, `-2` (blackhole) while being evaluated, so a thunk that demands itself aborts with `<<loop>>`, and `-3` (indirection) once the result is cached. `bhc_force` evaluates a thunk to weak head normal form, and `bhc_is_thunk` tests for one.
- **`Int` and `Double`** are stored directly in the pointer-sized slot (an `inttoptr`/bitcast, not a heap box). `Integer` is arbitrary precision and lives in the RTS.

Evaluation is mostly strict, with laziness where Haskell programs depend on it:

- A non-recursive `let` whose right-hand side does real work is compiled as a thunk and forced where the variable is used; literals, variables and lambdas are bound eagerly.
- A function argument or constructor field that can diverge (`error`, `undefined`, or a top-level binding whose body is one) is passed as a thunk, so `const 1 (error "boom")` and `fst (2, undefined)` behave as in GHC. This is what makes parsec's `many` work unmodified. Other arguments are evaluated before the call; full call-by-need for every argument is not implemented (see the lazy-arguments brief in the BHC source tree).
- Top-level values that refer to each other in a cycle (mutually recursive parser values, for example) are referenced through thunks; other top-level values are evaluated on first use.
- `Expr::Lazy` gives strict-profile code an explicit thunk.
- IO actions passed to combinators such as `catch`, `bracket` or `mask` are wrapped as thunks so they run only when the combinator runs them.

The LLVM module then goes through LLVM's own pass pipeline (`default<O1>` to `default<O3>`, following `-O`) and LLVM's code generator for the target. Calls in tail position are marked as tail calls.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     NATIVE CODEGEN (Core → LLVM IR)                         │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                         HEAP LAYOUTS                                     │
  │                                                                          │
  │   Constructor  ┌──────────┬──────────┬──────────┬─────┐                 │
  │                │ i64 tag  │ field 0  │ field 1  │ ... │                 │
  │                └──────────┴──────────┴──────────┴─────┘                 │
  │                                                                          │
  │   Closure      ┌──────────┬──────────────┬──────────────────┐           │
  │                │ fn_ptr   │ i64 env_size │ env[0..env_size] │           │
  │                └──────────┴──────────────┴──────────────────┘           │
  │                                                                          │
  │   Thunk        ┌──────────┬──────────┬──────────────┬─────────┐         │
  │                │ i64 tag  │ eval_fn  │ i64 env_size │ env[..] │         │
  │                └──────────┴──────────┴──────────────┴─────────┘         │
  │                                                                          │
  │   Thunk tags:   -1  unevaluated                                          │
  │                 -2  blackhole (being evaluated; cycle detection)         │
  │                 -3  indirection (result cached)                          │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                       FORCING A THUNK (bhc_force)                        │
  │                                                                          │
  │   tag == -1 ?  set tag = -2, call eval_fn(env), cache result, return it │
  │   tag == -2 ?  cyclic evaluation: abort with <<loop>>                    │
  │   tag == -3 ?  return the cached result                                  │
  │   otherwise    already a value: return it unchanged                      │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                       WHAT GETS THUNKED                                  │
  │                                                                          │
  │   let x = <non-trivial> in ...     thunk, forced at each use of x        │
  │   f a (error "..")                 bottom argument passed as a thunk     │
  │   K a undefined                    bottom field stored as a thunk        │
  │   lazy { e }                       explicit thunk (strict profiles)      │
  │   catch / bracket / mask actions   wrapped, run by the combinator        │
  │   everything else                  evaluated before use                  │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘
```

### Tensor IR and Loop IR (Numeric Profile)

When the profile is numeric (or `--tensor-fusion` is given), the driver also lowers Core to **Tensor IR** (`bhc-tensor-ir`). Tensor values carry their element type, shape and strides. Operations include element-wise `Map`, `ZipWith`, `Unary` and `Binary` ops, `Reduce`/`ReduceAll`/`Scan`/`Fold`, structural ops (`Reshape`, `Slice`, `Transpose`, `Broadcast`, `Concat`, `Split`), linear algebra (`MatMul`, `BatchMatMul`, `Dot`, `Outer`), `Conv`, `Gather`/`Scatter`, and constructors such as `Zeros`, `Ones`, `Full`, `Range` and `Eye`.

The fusion pass groups these operations into **kernels**, recording each fusion decision for the report. Kernels are lowered to **Loop IR** (`bhc-loop-ir`), which makes iteration explicit. Two analyses run over Loop IR: the vectorization pass finds innermost loops with sequential access that can use SIMD (with a scalar remainder loop, and FMA where available) and picks a vector width for the target architecture; the parallelization pass finds loops that can be split across workers, deterministically by default. Their results appear in the kernel report (`--kernel-report`).

> **Status:** Tensor IR, its fusion pass and the Loop IR analyses are tested in isolation and run during numeric builds. Their output reaches the WASM module, the GPU backends and the kernel report. Native LLVM code generation compiles the Core IR instead, so these kernels and their vectorization plans do not yet appear in native binaries.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  TENSOR IR → LOOP IR (NUMERIC PROFILE)                      │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                       TENSOR IR                                          │
  │                                                                          │
  │   TensorMeta = element dtype + Shape + Strides                           │
  │                                                                          │
  │   Element-wise:  Map, ZipWith, Unary, Binary                             │
  │   Reductions:    Reduce(axis), ReduceAll, Scan, Fold                     │
  │   Structure:     Reshape, Slice, Transpose, Broadcast, Concat, Split     │
  │   Linear alg.:   MatMul, BatchMatMul, Dot, Outer, Conv                   │
  │   Indexing:      Gather, Scatter                                         │
  │   Constructors:  Zeros, Ones, Full, Range, Eye, Constant                 │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼  fusion::fuse_ops
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                       KERNELS                                            │
  │                                                                          │
  │   sum (map (^2) t)   ═══►  one kernel: square and accumulate in one pass │
  │   map f (map g t)    ═══►  one kernel: apply f . g per element           │
  │                                                                          │
  │   each fusion decision recorded for --kernel-report                      │
  └─────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼  lower_kernels
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                       LOOP IR                                            │
  │                                                                          │
  │   for i in 0..n:                                                         │
  │     acc += t[i] * t[i]                                                   │
  │                                                                          │
  │   Vectorization analysis:   sequential innermost loop → SIMD width for   │
  │                             the target (x86-64, aarch64, WASM SIMD128),  │
  │                             scalar remainder, FMA where available        │
  │   Parallelization analysis: independent iterations → parallel chunks     │
  │                             (deterministic by default)                   │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘
```

### Code Generation Targets

The **LLVM backend** emits an object file for the host (or `--target`) triple, which the linker combines with the runtime. This is the main path and the one the test suites exercise most.

The **WASM backend** (`bhc-wasm`) lowers Core straight to a WebAssembly module and writes the binary itself: it emits the WASI imports (`fd_write`, `proc_exit`), a bump-allocated heap in linear memory, a small runtime for printing and allocation, and a `_start` entry point. No external WASM linker or `wasm-opt` step is involved. In numeric builds the Loop IR kernels are added to the module as extra functions, using SIMD128 where enabled. The crate also contains a mark-sweep collector for linear memory that the generated code does not call yet.

The **GPU backends** (`bhc-gpu`) compile Tensor IR kernels. Metal is the path that runs on real hardware: it generates Metal Shading Language, compiles it at run time, and executes on the GPU of an Apple-silicon Mac, with the kernel offload tested end to end on the device. The driver's GPU output today is PTX text for NVIDIA (`--emit ptx` or a CUDA target); the crate also has AMDGCN, SPIR-V and WGSL generators. CUDA and ROCm are validated in mock mode pending hardware.

The **linker** (`bhc-linker`) calls the system linker driver (`cc` on Linux and macOS) with the object file and BHC's static libraries: the RTS (`libbhc_rts`) and the stdlib's native parts (`libbhc_text`, `libbhc_base`, `libbhc_containers`).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          CODE GENERATION                                    │
└─────────────────────────────────────────────────────────────────────────────┘

                      Optimized Core                 Tensor IR kernels
                            │                       (numeric profile)
              ┌─────────────┴─────────────┐                 │
              ▼                           ▼                 ▼
       ┌─────────────┐             ┌─────────────┐   ┌─────────────┐
       │    LLVM     │             │    WASM     │   │     GPU     │
       │  (inkwell)  │             │  (bhc-wasm) │   │  (bhc-gpu)  │
       └─────────────┘             └─────────────┘   └─────────────┘
              │                           │                 │
              ▼                           ▼                 ▼
       ┌─────────────┐             ┌─────────────┐   ┌─────────────┐
       │  LLVM IR    │             │ WASM binary │   │ PTX (driver)│
       │             │             │             │   │ Metal (run- │
       │  - SSA form │             │  - WASI     │   │  time, M-   │
       │  - uniform  │             │    imports  │   │  series)    │
       │    ptr-size │             │  - bump     │   │ AMDGCN,     │
       │    values   │             │    heap     │   │ SPIR-V,WGSL │
       └─────────────┘             │  - _start   │   └─────────────┘
              │                    │  + Loop IR  │
              ▼                    │    kernels  │
       ┌─────────────┐             └─────────────┘
       │ LLVM passes │                    │
       │ default<On> │                    ▼
       │ + target    │               module.wasm
       │   codegen   │
       └─────────────┘
              │
              ▼
       ┌─────────────┐
       │   Linker    │
       │  cc + .o +  │
       │  libbhc_rts │
       │  libbhc_text│
       │  libbhc_base│
       │  libbhc_    │
       │  containers │
       └─────────────┘
              │
              ▼
         Executable
```

## Runtime System

The BHC runtime system (RTS) is a set of Rust crates under `rts/`: `bhc-rts` (the C-ABI entry points compiled code calls), `bhc-rts-alloc`, `bhc-rts-arena`, `bhc-rts-gc` and `bhc-rts-scheduler`. Compiled programs call into `bhc-rts` for allocation, thunk forcing, `Integer` arithmetic, text and container primitives, I/O, and exceptions (`exitWith`, a top-level handler for uncaught exceptions, and `catch`/`bracket`/`mask`).

### Memory Management

Compiled code allocates every heap object with `bhc_alloc`, which today is a direct call to the system allocator. Nothing is freed, so no pointer can dangle and short-lived batch programs (a compiler, a document converter) run correctly; a long-running server will grow without bound. The collector below exists to replace this.

> **Status:** `bhc-rts-gc` is a garbage-collector module with its data structures and unit tests, not a collector wired to compiled code. Compiled programs do not call it, codegen emits no GC roots or pointer maps, and its `minor_collect`/`major_collect` entry points are placeholders that record statistics without moving or freeing anything. Building a real collector has been deferred until a long-running target needs one.

What the module does provide:

- **Object headers** with a generation (nursery, survivor, old) and a pinned flag, and an allocation path that tracks bytes since the last collection against the nursery size.
- **Regions** for the design: a bump-allocated nursery (G0), a survivor generation (G1), an old generation (G2), and a pinned region for objects that must not move (FFI buffers, device memory).
- **A write barrier** that records old-to-young references in a remembered set.
- **An incremental tri-color marker** with a snapshot-at-the-beginning (SATB) write barrier, which splits marking into bounded increments to meet a pause budget. It is real code, tested over an abstract object graph.
- **Pause statistics** per collection kind.

Separately, `bhc-rts-arena` provides bump-allocated arenas with scoped lifetimes (`with_arena`, `with_scope`) and per-frame arenas for the realtime profile, and `bhc-rts-alloc` provides a static allocator for the embedded profile. Each profile has its own RTS configuration: the server profile asks for incremental marking with a 500 µs pause budget, realtime for a 1 ms bound and a smaller nursery, edge for minimal generation sizes, and embedded for no GC at all (enforced at compile time by escape analysis).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         MEMORY MANAGEMENT                                   │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                 WHAT COMPILED CODE DOES TODAY                            │
  │                                                                          │
  │   bhc_alloc(n)  =  system allocator, 8-byte aligned                      │
  │   (no threshold, no collection, no free: memory grows until exit)        │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                 bhc-rts-gc DESIGN  (module, not yet wired)               │
  │                                                                          │
  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
  │   │  Nursery     │  │  Survivor    │  │   Old Gen    │  │  Pinned    │ │
  │   │  (G0)        │  │  (G1)        │  │   (G2)       │  │  Region    │ │
  │   │              │  │              │  │              │  │            │ │
  │   │  - Bump      │  │  - Promoted  │  │  - Long-     │  │  - Never   │ │
  │   │    alloc     │  │    after one │  │    lived     │  │    moves   │ │
  │   │  - Frequent  │  │    survival  │  │  - Rare      │  │  - FFI,    │ │
  │   │    minor GC  │  │              │  │    major GC  │  │    DMA     │ │
  │   └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘ │
  │                                                                          │
  │   Write barrier: old → young stores recorded in a remembered set        │
  │                                                                          │
  │   Incremental marking (server / realtime configs):                       │
  │     white = unmarked, gray = to scan, black = scanned                    │
  │     SATB barrier keeps "no black → white" during mutation                │
  │     work done in bounded increments to meet the pause budget             │
  │                                                                          │
  │   minor_collect / major_collect: placeholders (stats only)               │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                 PROFILE CONFIGURATIONS                                   │
  │                                                                          │
  │   default    standard generation sizes                                   │
  │   server     incremental marking, 500 µs pause budget                    │
  │   numeric    smaller nursery                                             │
  │   edge       minimal nursery / survivor / old generation                 │
  │   realtime   incremental, 1 ms pause bound, per-frame arenas             │
  │   embedded   no GC; escaping allocations rejected at compile time        │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘
```

### Task Scheduler

`bhc-rts-scheduler` is a work-stealing task scheduler for structured concurrency, written in Rust on crossbeam deques. A pool of worker threads (one per core by default) each owns a local deque; new tasks go to a global injector queue, and an idle worker steals from the injector or from other workers' deques. Tasks are spawned inside a scope that outlives them, so every task finishes (or is cancelled) before its scope returns. Cancellation is cooperative and propagates to child tasks; tasks check for it at safe points, and deadlines bound how long an operation may run. Event hooks support tracing. The RTS also has an STM implementation (`TVar`, `TMVar`, `TQueue`).

Each profile picks a scheduler configuration: work stealing for server and numeric, a single worker for edge and embedded, and two workers without stealing (for deterministic scheduling) for realtime.

> **Status:** the scheduler and STM are tested from Rust. No compiled Haskell code reaches them yet: the stdlib declares the concurrency API (`spawn`, `await`, `withScope`, `atomically`), but codegen does not call into the scheduler, so compiled programs run on a single thread.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         TASK SCHEDULER                                      │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                                                                          │
  │     scope.spawn(task)                                                    │
  │            │                                                             │
  │            ▼                                                             │
  │     ┌──────────────────────────────────────────────────┐                 │
  │     │              Global injector queue                │                 │
  │     └──────────────────────────────────────────────────┘                 │
  │            │            │            │            │                      │
  │            ▼            ▼            ▼            ▼                      │
  │        ┌───────┐    ┌───────┐    ┌───────┐    ┌───────┐                  │
  │        │deque 0│    │deque 1│    │deque 2│    │deque 3│   ...            │
  │        └───────┘    └───────┘    └───────┘    └───────┘                  │
  │        worker 0     worker 1     worker 2     worker 3                   │
  │                                                                          │
  │        idle workers steal from the injector and from each other          │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                   TASK STATES (TaskState)                                │
  │                                                                          │
  │          spawn                                                           │
  │            │                                                             │
  │            ▼                                                             │
  │       ┌─────────┐    scheduled    ┌─────────┐                            │
  │       │ Pending │ ──────────────► │ Running │                            │
  │       └─────────┘                 └────┬────┘                            │
  │                                        │                                 │
  │              ┌─────────────────┬───────┴─────────┐                       │
  │              │ cancel          │ returns         │ panics                │
  │              ▼                 ▼                 ▼                       │
  │       ┌────────────┐     ┌───────────┐     ┌─────────┐                   │
  │       │ Cancelling │     │ Completed │     │ Failed  │                   │
  │       └─────┬──────┘     └───────────┘     └─────────┘                   │
  │             ▼                                                            │
  │       ┌───────────┐                                                      │
  │       │ Cancelled │                                                      │
  │       └───────────┘                                                      │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘
```

## Module System

BHC compiles modules separately or as a whole program, and the two meet at the interface file.

With `-c`, each source module compiles to an **object file (`.o`)**, written under `--odir`, and an **interface file (`.bhi`)**, written under `--hidir` at the module's path (`Data/Split.bhi`). The interface is a versioned binary file (magic `BHCI`) holding what an importer needs to type-check and lower against the module: a header with the module name and format version, the module's dependencies, exported values with their type schemes (type synonyms expanded so the interface stands alone), exported types and their constructors, classes, instances, instance method and class default implementations, pattern synonyms, and re-exports. Next to it the driver writes a **`.bhc` sidecar** with the module's lowered Core bodies and newtype definitions, which importing modules use for cross-module monad monomorphization.

**Import resolution** searches source roots first (the entry file's directory, `-I`, `--package-dir`, the stdlib path) and then compiled interfaces. A `--package-db` directory can be flat, as `bhc -c --hidir <db>` writes it, in which case any `.bhi` in it resolves; or it can hold registered packages as `<id>.conf` files (the layout hx produces), each listing its id, exposed modules, import directories and dependencies. A module resolves from a registered package only if that package exposes it, and `--package-id` narrows the visible packages to the given ids and their transitive dependencies.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          MODULE SYSTEM                                      │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                 SEPARATE COMPILATION  (bhc -c)                           │
  │                                                                          │
  │   Source Module (Data/Split.hs)                                          │
  │         │                                                                │
  │         ├──────────────► <odir>/Data/Split.o     compiled code           │
  │         │                                                                │
  │         ├──────────────► <hidir>/Data/Split.bhi  interface:              │
  │         │                   - header (name, version), dependencies       │
  │         │                   - exported values + type schemes             │
  │         │                   - exported types, constructors, classes      │
  │         │                   - instances, instance methods, defaults      │
  │         │                   - pattern synonyms, re-exports               │
  │         │                                                                │
  │         └──────────────► <hidir>/Data/Split.bhc  Core bodies + newtypes  │
  │                             (cross-module monomorphization)              │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    IMPORT RESOLUTION                                     │
  │                                                                          │
  │   import Data.List.Split (splitOn)                                       │
  │     1. Source roots: entry dir, -I, --package-dir, stdlib                │
  │     2. --hidir and flat --package-db dirs: <dir>/Data/List/Split.bhi     │
  │     3. Registered packages in --package-db (<id>.conf):                  │
  │          only packages that expose Data.List.Split,                      │
  │          scoped by --package-id (plus transitive depends)                │
  │     4. Load the interface, bind "splitOn" from its exports               │
  │                                                                          │
  │   Package DB entry (<id>.conf, written by hx)                            │
  │   ─────────────────────────────────────────                              │
  │     id:               split-0.2.5-...                                    │
  │     exposed-modules:  Data.List.Split, ...                               │
  │     import-dirs:      .../split-0.2.5                                    │
  │     depends:          base-...                                           │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘
```

## Compilation Modes

The `bhc` command compiles in one of these ways, depending on its arguments:

**Whole program from an entry file** (`bhc Main.hs`, or `bhc build Main.hs`) parses the entry file, discovers the modules it imports from the search paths, sorts them topologically, and compiles them in dependency order into one executable. Imported classes, instances and Core are visible across the whole program.

**Several files** (`bhc A.hs B.hs Main.hs`) are parsed to extract their imports, ordered topologically (circular imports are reported), and compiled in that order.

**Separate compilation** (`bhc -c Data/Split.hs --odir out --hidir db`) compiles one module to a `.o` and a `.bhi` without linking, resolving its imports from `--package-db`. This is how hx builds a package's modules from source into a package database.

**Checking only** (`bhc check File.hs`) runs the frontend and type checker without generating code; with `--package-dir` it can check a module against a dependency's source tree. `bhc run` compiles and runs a program.

Building a Cabal project with Hackage dependencies is hx's job: hx resolves and fetches dependencies, compiles them with `bhc -c` into a package database, then builds the project against it. `bhc` itself can still pull a single Hackage package's sources onto its search path with `--package name:version`.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         COMPILATION MODES                                   │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                                                                          │
  │   bhc Main.hs                          Whole program from entry file     │
  │        │                                                                 │
  │        ├── Parse Main.hs                                                 │
  │        ├── Discover imported modules on the search paths                 │
  │        ├── Topological sort                                              │
  │        ├── Compile each module (typecheck, Core, optimize, codegen)      │
  │        └── Link with RTS → executable                                    │
  │                                                                          │
  │   bhc A.hs B.hs Main.hs                Several files, dependency order   │
  │        │                                                                 │
  │        ├── Parse all, read imports                                       │
  │        ├── Topological sort (cycles reported)                            │
  │        └── Compile in order, link                                        │
  │                                                                          │
  │   bhc -c M.hs --odir o --hidir db      Separate compilation              │
  │        │                                                                 │
  │        ├── Resolve imports from --package-db / --hidir                   │
  │        ├── Compile to o/M.o                                              │
  │        └── Write db/M.bhi (+ db/M.bhc sidecar); no link                  │
  │                                                                          │
  │   bhc check M.hs                       Frontend + typecheck only         │
  │                                                                          │
  │   hx build                             Cabal projects (hx drives bhc -c) │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘
```

## Diagnostic System

BHC's diagnostics (`bhc-diagnostics`) follow the style of Rust's compiler: a coloured header with severity and code, the source location, a snippet with the offending span underlined, and notes and help lines.

A **diagnostic** has a severity (`Bug` for internal compiler errors, `Error`, `Warning`, `Note`, `Help`), an optional code, a message, labelled spans (one marked primary, any number of secondary ones), notes, and suggested fixes. Codes follow a scheme: `E00xx` for type mismatches and related errors, `E002x`/`E003x` for shape and tensor errors, `E004x` for pattern matching, `E005x` for modules and imports, and `W0xxx` for warnings; `bhc-diagnostics` holds a longer explanation for each code. Diagnostics can also be rendered as JSON or converted to LSP diagnostics for editors.

A type mismatch, for example, carries code `E0001`, labels the expression with the expected type, and adds a note that lines up the expected and found types:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DIAGNOSTIC SYSTEM                                    │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                      DIAGNOSTIC STRUCTURE                                │
  │                                                                          │
  │   ┌───────────────────────────────────────────────────────────────┐     │
  │   │  Diagnostic                                                    │     │
  │   │    ├── severity: Bug | Error | Warning | Note | Help           │     │
  │   │    ├── code: Option<"E0001", "W0010", ...>                     │     │
  │   │    ├── message: "type mismatch: ..."                           │     │
  │   │    ├── labels: [span + message, one primary]                   │     │
  │   │    ├── notes: [additional context]                             │     │
  │   │    └── suggestions: [fix-it hints]                             │     │
  │   └───────────────────────────────────────────────────────────────┘     │
  │                                                                          │
  │   Renderers: terminal (Cargo style), JSON, LSP                           │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                      EXAMPLE OUTPUT (shape)                              │
  │                                                                          │
  │   error[E0001]: type mismatch: expected `Int`, found `Bool`              │
  │     --> src/Main.hs:15:11                                                │
  │      |                                                                   │
  │   15 |       x = True                                                    │
  │      |           ^^^^ expected `Int`                                     │
  │      |                                                                   │
  │      = note: expected: Int                                               │
  │                 found: Bool                                              │
  │                                                                          │
  └─────────────────────────────────────────────────────────────────────────┘
```

## Further Reading

- [Quick Start](@/docs/quickstart.md) - Install BHC and write your first program
- [GHC Compatibility](@/docs/compatibility.md) - Supported Haskell features
- [Runtime Profiles](@/docs/profiles/_index.md) - Runtime profile details
- [Numeric performance](https://arcanist.sh/bhc/numeric/) - Tensor IR and the fusion contract
- [Targets](@/docs/targets/_index.md) - Backend targets (native, WASM)
