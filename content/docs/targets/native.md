+++
title = "Native Targets"
description = "Compiling to native executables"
weight = 1
+++

bhc compiles Haskell to native executables through LLVM, then links them with the system C toolchain (`cc`) against the bhc runtime libraries.

## Supported Platforms

| Platform | Architecture | Status |
|----------|--------------|--------|
| macOS | aarch64 (Apple silicon) | Release binaries, tested in CI |
| Linux | x86_64 | Release binaries, tested in CI |
| Other host platforms | - | No release binaries; building bhc from source may work but is untested |

Windows is not supported.

## Usage

By default, bhc compiles for the machine it runs on:

```bash
bhc Main.hs -o main
./main
```

Without `-o`, the executable is named after the source file (`Main.hs` becomes `Main`).

### Cross-Compilation

Linking an executable for another platform is not supported yet. What works today is emitting an object file for another architecture with `-c` and a full target triple:

```bash
# x86_64 Linux object file, produced on any host
bhc --target x86_64-unknown-linux-gnu -c Main.hs --odir build

# RISC-V 64 object file
bhc --target riscv64-unknown-linux-gnu -c Main.hs --odir build
```

The triple must have at least three parts (`arch-vendor-os`). A shorter or unrecognized triple such as `x86_64-linux` currently falls back to the host target without an error, so always spell the triple out.

## Optimization Levels

`-O` takes a level from 0 to 3. The default is `-O2`.

```bash
# No optimization
bhc -O0 Main.hs -o main

# Light optimization
bhc -O1 Main.hs -o main

# Default
bhc -O2 Main.hs -o main

# Aggressive
bhc -O3 Main.hs -o main
```

## Separate Compilation

`-c` compiles a module to an object file without linking. `--odir` sets where the object goes; adding `--hidir` also writes the module's interface (`.bhi`):

```bash
bhc -c Main.hs --odir build --hidir build
# build/Main.o, build/Main.bhi
```

Interfaces in a directory can then be consumed by later builds with `--package-db <dir>`.

## Linking

bhc links each program against its runtime libraries (`bhc_rts`, `bhc_base`, `bhc_text`, `bhc_containers`) plus the platform's system libraries. There are no `--static` or `--dynamic` flags: a release installation ships the runtime as static archives, so the resulting executable has no dependency on bhc at run time.

When running a development build of bhc from a source checkout, point `LIBRARY_PATH` at the build directory so the linker finds the runtime libraries:

```bash
LIBRARY_PATH=/path/to/bhc/target/release bhc Main.hs -o main
```

## Platform-Specific Notes

### macOS

Executables link against `libSystem`. Universal (x86_64 + aarch64) binaries are not supported.

### Linux

Executables link against glibc together with `pthread`, `dl` and `m`.
