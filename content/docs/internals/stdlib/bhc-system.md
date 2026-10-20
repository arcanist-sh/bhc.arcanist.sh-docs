+++
title = "bhc-system"
description = "System interfaces"
weight = 8
+++

# bhc-system

Operating-system interfaces for the Basel Haskell Compiler.

## Overview

`System.IO`, `System.Environment`, `System.Exit`, `System.Directory` and `System.FilePath` work in compiled programs today. They are built into the compiler, and the C functions behind them (`bhc_open_file`, handle operations, `bhc_get_args`, `bhc_get_env`, `bhc_create_directory`, `bhc_list_directory`, `bhc_take_extension`, `bhc_exit`, ...) are implemented in [bhc-rts](@/docs/internals/runtime/bhc-rts.md) (`rts/bhc-rts/src/ffi.rs`), which every program links.

The `bhc-system` crate is a separate, more structured Rust implementation of the same areas, plus process spawning, with the intended Haskell API in `hs/BHC/System/{IO, Environment, Exit, Directory, FilePath, Process}.hs`. It is not linked into programs yet, and generated code does not call it.

## Rust modules (`bhc-system`)

| Module | Contents |
|--------|----------|
| `io` | File handles and buffered IO |
| `environment` | Environment variables and program arguments |
| `filepath` | Path manipulation |
| `directory` | Directory operations |
| `exit` | Exit codes |
| `process` | Process spawning and waiting |

## What works today

Checked against GHC 9.8.2 on 2026-10-20, one expression per program:

```haskell
-- System.IO
openFile, hPutStrLn, hGetLine, hClose, hSetBuffering
readFile, writeFile, appendFile
stdin, stdout, stderr, isEOF

-- System.Environment
getArgs, getProgName

-- System.Exit
exitWith (ExitFailure n)   -- process exit status is n
exitSuccess

-- System.Directory
doesFileExist, doesDirectoryExist, createDirectory, removeDirectory,
listDirectory, removeFile, renameFile, copyFile, getCurrentDirectory

-- System.FilePath
(</>), takeFileName, takeDirectory, takeExtension, takeBaseName,
replaceExtension, dropExtension, splitExtension, splitPath, isAbsolute
```

## Known gaps

- `getEnv` crashes (segmentation fault), and `lookupEnv` results print nothing. `setEnv` is a stub that aborts.
- `withFile` and `createDirectoryIfMissing` are stubs that abort.
- `System.Process` (`callCommand`, `readProcess`, ...) is not implemented in compiled programs; its functions are stubs that abort.
- `doesFileExist p >>= print` prints a pointer; `do { b <- doesFileExist p; print b }` prints `True`.
- There is no networking module.
