+++
title = "GHC Compatibility"
description = "bhc compatibility with GHC and Haskell standards"
weight = 3
+++

bhc aims to compile ordinary Haskell source unchanged. Its largest test is Pandoc: all 221 Pandoc library modules pass `bhc check` and compile, and `readMarkdown` and `writeHtml5String` run end to end. This page lists what that coverage looks like extension by extension, and where results still differ from GHC.

## Language Standards

| Standard | Status |
|----------|--------|
| Haskell 2010 | The target. The core language parses, type-checks and compiles, including the layout rule, type classes, records, guards and the module system. A few runtime results still differ from GHC (see [Known differences](#known-differences)). |
| GHC2021 / GHC2024 | Not selectable as a language set. Most of their individual extensions are accepted or always on (below). |

`{-# LANGUAGE Haskell2010 #-}` and `{-# LANGUAGE GHC2021 #-}` currently produce an "unknown extension" warning and are otherwise ignored. The `--edition` flag is accepted on the command line but does not change how code is compiled yet.

## GHC Extensions

bhc reads every `LANGUAGE` pragma and sorts each name into one of three groups (`crates/bhc-ast/src/lib.rs`, `Extension::status`). Most extensions are always on in bhc's parser and type checker, so the pragma does not switch anything; it is accepted so that GHC code compiles unchanged.

### Accepted and implemented

Type system: `GADTs`, `TypeFamilies`, `DataKinds`, `KindSignatures`, `RankNTypes`, `ExistentialQuantification`, `ScopedTypeVariables`, `TypeApplications`, `FlexibleInstances`, `FlexibleContexts`, `MultiParamTypeClasses`, `FunctionalDependencies`, `TypeOperators`, `ExplicitForAll`.

Syntax: `LambdaCase`, `MultiWayIf`, `BlockArguments`, `PatternGuards`, `ViewPatterns`, `PatternSynonyms`, `RecordWildCards`, `NamedFieldPuns`, `OverloadedStrings`, `TupleSections`, `OverloadedLists`, `NumericUnderscores`, `BinaryLiterals`, `NegativeLiterals`.

Strictness: `BangPatterns`, `StrictData`, `Strict`.

Deriving: `DeriveFunctor`, `DeriveFoldable`, `DeriveTraversable`, `DeriveGeneric`, `DeriveDataTypeable`, `DerivingStrategies`, `DerivingVia`, `GeneralizedNewtypeDeriving`, `StandaloneDeriving`.

Other: `ForeignFunctionInterface`, `CPP`, `EmptyDataDecls`, `EmptyCase`, `InstanceSigs`, `DefaultSignatures`, `ExplicitNamespaces`, `UndecidableInstances`, `OverlappingInstances`.

"Accepted" means the syntax compiles; it does not mean every program using the extension behaves exactly as under GHC. The cases we know about are listed under [Known differences](#known-differences).

### Accepted without being enforced

These are accepted silently for compatibility. Their syntax parses, but bhc does not implement the extra checking or representation they imply:

`MonoLocalBinds`, `PolyKinds`, `TypeInType`, `StarIsType`, `StandaloneKindSignatures`, `RoleAnnotations`, `AllowAmbiguousTypes`, `TypeSynonymInstances`, `ImpredicativeTypes`, `TypeFamilyDependencies`, `QuantifiedConstraints`, `DeriveAnyClass`, `NoImplicitPrelude`, `NoMonomorphismRestriction`, `DisambiguateRecordFields`, `DuplicateRecordFields`, `NoFieldSelectors`, `ApplicativeDo`, `NumDecimals`, `PackageImports`, `ImportQualifiedPost`, `UnicodeSyntax`, `ParallelListComp`, `TransformListComp`, `MonadComprehensions`, `ExtendedDefaultRules`, `PostfixOperators`, `Arrows`.

`LinearTypes` is accepted the same way: `a %1 -> b` parses and compiles, but linearity is not checked. `MagicHash` and `UnboxedTuples` are lexed and parsed, and `MagicHash` code such as `I# (x +# 1#)` compiles using boxed values; unboxed tuples do not compile to working code yet.

### Recognised, not yet implemented

bhc prints `extension ... is not yet implemented and will be ignored` for:

- `TemplateHaskell`, `TemplateHaskellQuotes`: there is no compile-time `Q` evaluator. A small set of well-known declaration splices is expanded directly by the compiler: `$(makeLenses ''T)` (and `makeLensesFor`) generate van Laarhoven lenses for the record fields of `T` (`crates/bhc-lower/src/th_expand.rs`). Other splices are dropped.
- `QuasiQuotes`
- `ConstraintKinds` (simple constraint synonyms such as `type C a = (Show a, Num a)` still work)
- `DeriveLift`, `HexFloatLiterals`, `CApiFFI`, `UnliftedFFITypes`, `NamedDefaults`

Any other extension name, for example `UnboxedSums`, produces an "unknown extension" warning.

## Known differences

These programs compile but give a different result from GHC with the current build. They are tracked as bugs:

- Showing a value through a class dictionary carried by an existential constructor prints the wrong output for non-`Int` types.
- `fmap`, `foldr` and `traverse` through derived `Functor`, `Foldable` and `Traversable` instances on user-defined types produce wrong values, and `sum`/`length` type-check only at lists.
- Associated type families whose instances are used at a literal (`insert 1 (empty :: IntBag)` with `type Elem IntBag = Int`) fail to type-check.
- A GADT match that refines the result type to more than one index type (`eval :: Expr a -> a` over `Expr Int` and `Expr Bool`) fails to type-check; single-index GADTs work.
- Arithmetic on `Double` values in lists (`sum`, folds) and on `Double` fields bound by a constructor pattern can give wrong results, and `[1..10] :: [Double]` does not compile.
- `Map.toList` returns keys in insertion order instead of ascending order.
- `pi` is not in scope from the Prelude.

## Base Library

bhc ships its own `Prelude` and standard library (`stdlib/` in the compiler repository) covering the commonly used parts of `base`, `containers`, `text`, `mtl`/`transformers` and file IO. It is not a complete copy of GHC's `base`: names that bhc does not provide are reported as unbound or replaced by a stub that warns at compile time.

## Hackage Packages

`bhc --package name:version` fetches a package from Hackage, `--package-dir` points at dependency source trees, and `--package-db` reads compiled `.bhi` interfaces. Pandoc, parsec and blaze-html are the packages exercised most. Whether another package works depends on:

1. The extensions it uses (see above)
2. Whether it relies on GHC internals (`GHC.Prim`, unboxed types, primops)
3. Whether it needs Template Haskell beyond the supported splices

## Reporting Compatibility Issues

If you encounter compatibility issues, please report them on [GitHub](https://github.com/arcanist-sh/bhc/issues) with:

1. The code that fails to compile
2. The error message from bhc
3. Whether the code compiles with GHC
