+++
title = "bhc-prelude"
description = "Standard prelude"
weight = 1
+++

# bhc-prelude

The Prelude for the Basel Haskell Compiler.

## Overview

The Prelude that every module imports implicitly is **built into the compiler**. Its types, classes and functions are defined in the builtin tables of `bhc-lower` (`define_builtins` in `crates/bhc-lower/src/context.rs`) and `bhc-typeck`, and `bhc-codegen` implements them, partly inline and partly as calls into `libbhc_rts.a` and `libbhc_base.a`. The implicit import is skipped for modules named `Prelude` or `BHC.Prelude` and for modules with `{-# LANGUAGE NoImplicitPrelude #-}`.

The `bhc-prelude` crate itself is intentionally empty on the Rust side. Its `hs/BHC/Prelude.hs` is a Haskell description of the intended Prelude (export list, class hierarchy, list functions); the compiler does not load it when building programs.

## Core Types

| Type | Notes |
|------|-------|
| `Bool`, `Ordering`, `()` | |
| `Char` | Unicode code point |
| `Int` | 64-bit |
| `Integer` | Arbitrary precision, via `num-bigint` in `bhc-rts` |
| `Double`, `Float` | |
| `Rational` | Heap-allocated numerator/denominator pair in `bhc-rts` |
| `Maybe a`, `Either a b`, `[a]`, tuples | |
| `IO a`, `IORef a` | |

## Classes

`Eq`, `Ord`, `Show`, `Read`, `Enum`, `Bounded`, `Num`, `Integral`, `Fractional`, `Floating`, `Real`, `RealFrac`, `Semigroup`, `Monoid`, `Functor`, `Applicative`, `Monad`, `Foldable` and `Traversable` are known to the type checker. Derived `Show`, `Eq` and `Ord` work for user data types, as do `succ` on a derived `Enum`; user-defined classes and instances are dictionary-passed.

## What works today

Checked against GHC 9.8.2 on 2026-10-20, one expression per program:

- List functions: `map`, `filter`, `foldr`, `foldl`, `sum`/`product` at `Int`, `zip`, `unzip`, `lookup`, `elem`, `takeWhile`, `span`, `replicate`, `concatMap`, `words`, `unwords`, `lines`, `unlines`, `take` of `iterate` and `cycle`, `scanl`, `until`
- Numeric: `div`, `mod`, `gcd`, `lcm`, `sqrt`, `/` and `fromIntegral` at `Double`, `truncate`, `2 ^ 64 :: Integer`
- `show` and `print` for `Int`, `Integer`, `Double`, `Bool`, `String`, lists, `Maybe`, `Either`, pairs and derived `Show` instances
- `read` at `Int`, `reads`
- `fmap` over lists and `Maybe`, `pure`, `mapM_`, `foldM`, `when`, `forM_`, `replicateM` in `IO`
- `IO`: `putStrLn`, `print`, `getContents`, `readFile`, `writeFile`, `appendFile`

## Known gaps

Also checked on 2026-10-20 against GHC:

| Expression | bhc today |
|------------|-----------|
| `pi` | Unbound variable (compile error) |
| `sum [1, 2, 3 :: Integer]`, `product [1..10 :: Integer]` | Runtime crash |
| `sum [1.5, 2.5 :: Double]` | Prints `NaN` |
| `2 ** 10 :: Double` | Aborts: `stub: ** not implemented` |
| `area (Rect w h) = w * h` with `Double` fields | Wrong result (`0.0` for `Rect 2.0 3.0`) |
| `(+) <$> Just 1 <*> Just 2` | Runtime error in partial application |
| `sequence [Just 1, Just 2]`, `traverse Just [1, 2]` | Wrong result (`Just [1]`, a pointer) |
| `print (Just 1 >>= \x -> Just (x + 1))` | Prints a pointer |
| `fmap (+1) (Right 1)` | Runtime crash |
| `do { l <- getLine; putStrLn l }` | Segmentation fault (`getLine` then `length l` works; `getContents` works) |
| `[minBound .. maxBound :: Color]`, `map fromEnum [Red, Blue]` on a derived `Enum` | Prints pointers |
| `print` of 3-tuples, nested tuples, `zip3` results | Prints a pointer |
| `print (chr 66)`, `print (maximum "hello")` | Prints a number, or crashes, where a `Char` comes back from a polymorphic function |
| `return True >>= print` | Prints a pointer (`do { b <- ...; print b }` works) |
| `show (Just (-3))` | `"Just -3"` (GHC: `"Just (-3)"`) |
| `read "3.25" :: Double`, `read "[1,2,3]" :: [Int]` | `no parse` |
| Record syntax in derived `Show` | Printed positionally: `P 1 2` |

The [project status page](https://arcanist.sh/bhc/status/) tracks measured coverage over time.
