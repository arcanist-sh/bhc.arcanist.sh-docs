+++
title = "Numeric Profile"
description = "Optimized for computational and numeric workloads"
weight = 3
+++

The **numeric** profile turns on BHC's list-fusion rewrites and its Tensor IR pipeline. Today the rewrites remove intermediate lists for a set of common fold patterns over `Int`. The wider design (unboxed numerics, SIMD, tensor kernels in native code) is described at the end of this page and is not yet realised in native binaries.

## When to Use

Use the numeric profile when:

- Your hot loops are folds over `Int` ranges or chains of `map`
- You want to inspect the Tensor IR and kernel report for a program
- You are working on BHC's numeric pipeline itself

## Usage

```bash
bhc --profile=numeric Compute.hs -o compute
```

## What it does today

### List fusion

These Core-to-Core rewrites run only in the numeric profile (`crates/bhc-core/src/simplify/fuse.rs`):

| Pattern | Rewritten to |
|---------|--------------|
| `sum [a..b]`, `product [a..b]`, `foldl' op z [a..b]` on `Int` | a counting loop with no list |
| the same, through an `Int -> Int` `map f` | a counting loop with no list |
| `map f (map g xs)` | `map (\v -> f (g v)) xs` |
| `sum (map f xs)` with a named `f :: Int -> Int` | a single strict `foldl'` |
| `foldl' op z (map f xs)` | a single strict `foldl'` |

For example, this program compiles to a loop that never builds the list, and runs much faster than the same program under the default profile:

```haskell
main :: IO ()
main = print (sum (map (*2) [1..30000000 :: Int]))
```

The counting-loop rewrites apply to `Int` ranges only. `sum (map f xs)` does not fuse yet when `f` is a lambda; `foldl'` and `map` chains fuse for any function.

### Tensor IR and the kernel report

The numeric profile lowers each module to Tensor IR and runs the Tensor and Loop IR fusion passes. You can print a report of the result:

```bash
bhc --profile=numeric --kernel-report Compute.hs
```

The Tensor IR recognises tensor operations, not ordinary list code, so for list programs the report shows 0 kernels. The fused Loop IR feeds the WebAssembly and GPU backends. Native code is generated from Core, where the list rewrites above apply.

### Other differences from the default profile

- Demand analysis and worker/wrapper, which the default profile runs, are skipped
- WebAssembly output enables SIMD128 and uses a 2 MB initial / 32 MB maximum memory

## Planned

The numeric profile is designed to provide, and does not yet provide in native code:

- **Strict evaluation in hot paths.** Compiled code is lazy under every profile today.
- **Unboxed numerics.** Native code boxes `Int` and `Double` values.
- **SIMD and parallel loops.** The Loop IR has vectorisation and parallelisation passes; native code generation does not consume them yet.
- **Tensor and vector libraries.** `BHC.Numeric.Tensor`, `BHC.Numeric.Vector` and `BHC.Numeric.Matrix` are drafted in `stdlib/bhc-numeric`, but their functions are not available to compiled programs yet, and neither is `Data.Vector.Unboxed`.
- **GPU offload.** The Metal runtime runs Tensor IR kernels on Apple-silicon GPUs from Rust; a compiled `.hs` program does not offload automatically yet.

Progress is tracked on the [status page](https://arcanist.sh/bhc/status/).
