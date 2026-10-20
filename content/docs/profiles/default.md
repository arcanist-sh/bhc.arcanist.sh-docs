+++
title = "Default Profile"
description = "General-purpose compilation profile"
weight = 1
+++

The **default** profile is the standard compilation mode for bhc. It uses ordinary lazy Haskell semantics and is the profile the test suites and the GHC differential harness run against.

## When to Use

Use the default profile when:

- Building general-purpose applications
- You want the best-tested path through the compiler
- You don't have specific performance requirements
- Testing or prototyping

## Usage

The default profile is used automatically:

```bash
bhc Main.hs -o main
```

Or explicitly:

```bash
bhc --profile=default Main.hs -o main
```

## Characteristics

- **Evaluation**: Lazy (standard Haskell semantics); infinite lists such as `takeWhile (< 10) [1..]` work
- **Optimisation**: Core simplifier, demand analysis with worker/wrapper, and dictionary specialisation
- **Memory**: Compiled programs allocate without collecting. A garbage collector exists as a tested runtime module but is not yet connected to compiled code, so long-running programs grow in memory.
- **Concurrency**: `forkIO` is not implemented in native code yet; programs are single-threaded
- **FFI**: `foreign import ccall` works for numeric types such as `Int` and `Double`

## Example

```haskell
module Main where

import Data.List (sort)

main :: IO ()
main = do
    let numbers = [3, 1, 4, 1, 5, 9, 2, 6]
    putStrLn "Sorted list:"
    print (sort numbers)

factorial :: Integer -> Integer
factorial 0 = 1
factorial n = n * factorial (n - 1)
```

Compile and run:

```bash
bhc Main.hs -o main
./main
```

Output:

```
Sorted list:
[1,1,2,3,4,5,6,9]
```
