+++
title = "bhc-utils"
description = "Utility functions"
weight = 9
+++

# bhc-utils

Time, random-number and JSON primitives for the Basel Haskell Compiler.

## Overview

This crate provides Rust implementations of date and time handling, a random-number generator and a JSON parser, with a small C interface. The intended Haskell API is in `hs/BHC/Data/{Time, Random, JSON}.hs`.

**Status:** the Rust code is implemented and unit-tested, but it is not yet connected to compiled Haskell: `libbhc_utils.a` is not on the native link line and generated code calls none of its functions. In compiled programs today, `Data.Time.Clock.getCurrentTime` returns without a real time value (`print` shows a pointer) and `System.Random.randomRIO` aborts as a stub.

## Rust modules

| Module | Contents |
|--------|----------|
| `time` | `Duration`, `Instant`, `Date`, `Time`, `DateTime`; `sleep`, `measure` |
| `random` | `Rng` (seedable), `random_u64`, `random_f64`, `random_bool`, `weighted_choice`, `Normal` distribution |
| `json` | `Json` value type, parser, `JsonError` |

## C interface

| Area | Functions |
|------|-----------|
| Time | `bhc_time_now`, `bhc_sleep_millis` |
| Random | `bhc_rng_new`, `bhc_rng_from_seed`, `bhc_rng_next_u64`, `bhc_rng_range`, `bhc_rng_free` |
| JSON | `bhc_json_parse`, `bhc_json_free` |

## Other utility modules

`Debug.Trace`, `System.IO.Unsafe`, `Data.Function`, `Data.Bifunctor`, `Data.Coerce` and `GHC.Generics` are not part of this crate; like the rest of the base library they are built into the compiler. Their state on 2026-10-20, checked against GHC 9.8.2:

| Module | Works | Gaps |
|--------|-------|------|
| `Data.Function` | `(&)` | ``(+) `on` length`` prints nothing; `print (fix f 5)` prints a pointer |
| `Data.Bifunctor` | `first` | `bimap` is a stub that aborts |
| `Debug.Trace` | | `trace` is a stub that aborts |
| `System.IO.Unsafe` | | `unsafePerformIO` is a stub that aborts |
| `Data.Coerce` | | `coerce` is a stub that aborts |
| `GHC.Generics` | The import is accepted | |
