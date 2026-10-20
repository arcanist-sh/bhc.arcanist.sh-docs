+++
title = "Quick Start"
description = "Compile and run your first Haskell program with BHC, pick a runtime profile, build for WebAssembly, and build a whole project through hx."
weight = 2
+++

This guide assumes BHC is [installed](@/docs/installation.md) and `bhc --version` works.

## Your first program

Create `Main.hs`:

```haskell
module Main where

main :: IO ()
main = putStrLn "Hello from BHC!"
```

Compile it to a native executable and run it:

```bash
bhc Main.hs -o hello
./hello
```

```
Hello from BHC!
```

BHC compiles through LLVM, so `hello` is an ordinary native binary with the BHC runtime linked in.

## Check, run, report

```bash
bhc check Main.hs                              # type-check only, no code generation
bhc run Main.hs                                # run with the Core interpreter
bhc --profile=numeric --kernel-report Main.hs  # print the numeric profile's kernel report
```

## Something with more Haskell

Data types, a type class, `Data.Map` and `do` notation:

```haskell
module Main where

import qualified Data.Map as Map

data Shape = Square Int | Rect Int Int
  deriving (Show, Eq)

class HasArea a where
  area :: a -> Int

instance HasArea Shape where
  area (Square s) = s * s
  area (Rect w h) = w * h

main :: IO ()
main = do
  let shapes = [("tile", Square 3), ("door", Rect 1 2)]
      byName = Map.fromList shapes
  mapM_ (\(name, s) -> putStrLn (name ++ ": " ++ show (area s))) shapes
  print (Map.lookup "door" byName)
  print (sum (map (area . snd) shapes))
```

```bash
bhc Main.hs -o shapes && ./shapes
```

```
tile: 9
door: 2
Just (Rect 1 2)
11
```

## Choose a runtime profile

Profiles keep the language and change the runtime contract. The default is the everyday lazy path:

```bash
bhc --profile=numeric Compute.hs -o compute   # tensor pipeline, kernel reports
bhc --profile=server Server.hs -o server      # structured concurrency runtime
bhc --profile=edge Fn.hs -o fn                 # minimal runtime
```

See [Runtime Profiles](@/docs/profiles/_index.md) for what each one changes, and the [status page](https://arcanist.sh/bhc/status/) for how much of each is wired into compiled code today.

## Build for WebAssembly

```bash
bhc --target wasm32-wasi Main.hs -o hello.wasm
wasmtime hello.wasm
```

The WebAssembly backend emits WASI modules directly, without LLVM. See [WebAssembly](@/docs/targets/wasm.md) for file IO and what the backend covers.

## Build a project with hx

For a project with a `.cabal` file and Hackage dependencies, let [hx](https://arcanist.sh/hx/) drive BHC. It resolves the dependencies, builds them from source with BHC, and links your executables:

```bash
hx build --backend bhc
hx test --backend bhc
```

To make BHC the project's compiler, set it in `hx.toml`; the command-line flag still wins when you pass it:

```toml
[compiler]
backend = "bhc"

[compiler.bhc]
profile = "numeric"          # default, server, numeric or edge
target = "wasm32-wasi"       # optional cross-compilation target
emit_kernel_report = true    # print the numeric profile's kernel report
tensor_fusion = true         # force the tensor fusion pipeline on
```

hx can also pin a curated [BHC Platform](https://docs.arcanist.sh/hx/) snapshot, a set of package versions known to build with BHC:

```bash
hx bhc-platform list
hx bhc-platform set bhc-platform-2026.1
```

## Next steps

- [Runtime Profiles](@/docs/profiles/_index.md) and [Targets](@/docs/targets/_index.md)
- [GHC Compatibility](@/docs/compatibility.md): extensions and known gaps
- [Internals](@/docs/internals/_index.md): how the compiler is put together
- The [API reference](https://docs.arcanist.sh/bhc/api/) for the Prelude and standard library
- Try BHC without installing anything in the [playground](https://arcanist.sh/bhc/playground/)
