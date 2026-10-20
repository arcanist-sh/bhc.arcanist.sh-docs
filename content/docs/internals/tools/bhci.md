+++
title = "bhci"
description = "Interactive REPL"
weight = 2
+++

# bhci

Interactive Read-Eval-Print Loop for the Basel Haskell Compiler.

## Overview

`bhci` evaluates Haskell expressions interactively. Each input is parsed, type checked, lowered to Core and run by bhc's Core interpreter, the same evaluator behind `bhc run` and the [playground](@/docs/internals/tools/bhc-playground.md). It does not use the native code generator, so its coverage of the standard library is narrower than compiled code.

## Starting the REPL

```bash
# Start REPL
bhci

# Load a file on startup
bhci Main.hs

# Choose a profile
bhci --profile=numeric

# Resolve imports from a package database
bhci --package-db build/db --package-id containers
```

`bhc repl` starts the same REPL.

## REPL Commands

| Command | Description |
|---------|-------------|
| `:load FILE` (`:l`) | Load a module |
| `:reload` (`:r`) | Reload the loaded modules |
| `:type EXPR` (`:t`) | Show the type of an expression |
| `:kind TYPE` (`:k`) | Show the kind of a type |
| `:info NAME` (`:i`) | Show the type and origin of a value |
| `:browse [MODULE]` | List bindings in scope |
| `:show [bindings\|imports\|modules\|packages]` | Show REPL state |
| `:module [+/-] MOD` | Add or remove modules from scope |
| `:set +t` / `:set +s` | Show types / timing after each evaluation |
| `:set profile NAME` | Change the profile |
| `:cd DIR` | Change working directory |
| `:!CMD` | Run a shell command |
| `:quit` (`:q`) | Exit REPL |

`import` statements and `let name = expr` bindings can be entered directly.

## Interactive Session

```
bhci:001> map (+1) [1,2,3]
[2,3,4]
  :: [Int]
bhci:002> let y = 21
y :: Int = 21
bhci:003> y * 2
42
  :: Int
bhci:004> :kind Maybe
Maybe :: * -> *
bhci:005> :load Square.hs
[1 of 1] Compiling Square.hs...
Ok, 2 definitions loaded from module Main.
bhci:006> square 9
81
  :: Int
```

## Multi-line Input

```
bhci:001> :{
bhci:002| let z = 5
bhci:003| :}
z :: Int = 5
```

## Current Limitations

- `let` at the prompt binds a single name (`let f = \x -> x + 1`); the function form `let f x = ...` is not accepted yet. Define functions in a file and `:load` it.
- `:info` does not describe type classes yet.
- Some library functions are not evaluated by the interpreter yet; `sort [3,1,2]`, for example, prints `<<primop: Sort (partial)>>` instead of `[1,2,3]`. Compiled code handles them.
- Inferred types are printed with internal type-variable names (`map :: ((t560 -> t561) -> ...)`).
- Output from `IO` actions, including a loaded module's `main`, is currently printed twice.
- There is no debugger (`:trace`, `:step`, breakpoints).
