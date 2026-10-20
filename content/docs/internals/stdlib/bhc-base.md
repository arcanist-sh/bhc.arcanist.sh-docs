+++
title = "bhc-base"
description = "Base library"
weight = 2
+++

# bhc-base

Base library support for the Basel Haskell Compiler.

## Overview

`bhc-base` provides the Rust primitives behind `Data.Char` and the heavier `Data.List` and string functions. It is built as `libbhc_base.a`, which the native backend links into every program, and generated code calls its functions directly.

The base modules themselves (`Data.List`, `Data.Char`, `Data.Maybe`, `Control.Monad`, `Control.Exception`, ...) are built into the compiler, like the [Prelude](@/docs/internals/stdlib/bhc-prelude.md). File and handle IO, exceptions and `IORef` are implemented in `bhc-rts`, not here.

## Rust modules

| Module | C functions |
|--------|-------------|
| `char` | `bhc_char_is_alpha`, `is_digit`, `is_alphanumeric`, `is_space`, `is_upper`, `is_lower`, `is_print`, `is_control`, `is_hex_digit`, `is_oct_digit`, `is_punctuation`, `is_symbol`, `is_separator`, `is_letter`, `is_number`, `is_ascii`, `is_latin1`, ...; `to_upper`, `to_lower`, `digit_to_int`, `int_to_digit`, `ord`, `chr` |
| `list` | `bhc_list_sort`, `sort_by`, `sort_on`, `nub`, `nub_by`, `group`, `group_by`, `intercalate`, `transpose`, `insert`, `delete_by`, `union_by`, `intersect_by`, `strip_prefix`, `map_accum_l`, `map_accum_r` |
| `string` | `bhc_string_lines`, `unlines`, `words`, `unwords`; `bhc_read_int`, `bhc_try_read_int` |

Character classification uses Rust's `char` methods, so it follows Unicode.

## Haskell sources

`hs/BHC/` contains `BHC.Data.{Char, Either, Function, List, Maybe, Ord, String, Tuple}` and `BHC.Control.{Applicative, Arrow, Category, Exception, Monad}`. These describe the intended library; they are not compiled into programs.

## What works today

Checked against GHC 9.8.2 on 2026-10-20, one expression per program:

- `Data.List`: `sort`, `sortBy`, `sortOn`, `nub`, `group`, `intercalate`, `transpose`, `tails`, `partition`, `insert`, `delete`, `intersect`, `isPrefixOf`, `isSuffixOf`, `isInfixOf`, `stripPrefix`, `foldl'`, `foldl1`, `splitAt`, `findIndex`, `elemIndex`
- `Data.Char`: `toUpper`, `toLower`, `isDigit`, `isSpace`, `ord`
- `Data.Maybe`: `fromMaybe`, `mapMaybe`, `catMaybes`
- `Control.Monad`: `when`, `unless`, `forM_`, `mapM_`, `foldM`, `replicateM`
- `Control.Exception`: `catch` (with `SomeException` or `IOException` handlers), `bracket`, `finally`
- `Data.IORef`: `newIORef`, `readIORef`, `modifyIORef`, `atomicModifyIORef'`

## Known gaps

- `subsequences`, `permutations`, `bracket_`, `evaluate`, `withFile`, `Data.Ord.Down` and the `ErrorCall` constructor are stubs that abort when run.
- `union [1,2] [2,3]` returns `[1,2,2,3]` (GHC: `[1,2,3]`).
- `iterate (*2) 1 !! 10` aborts with `!!: index too large`.
- `print` of a nested list returned by `group`, `transpose`, `tails` or `inits` prints pointers, although the values are right (`map length (group xs)` and `map sum (transpose xss)` match GHC). `print` of a caught exception (`show (e :: IOException)`) and of `sortBy (comparing snd)` over pairs also prints pointers.
- ``groupBy ((==) `on` f)`` prints nothing.
- `try` around `ioError` prints `<<exception>>` instead of returning `Left`.
