+++
title = "bhc-package"
description = "Package management"
weight = 63
+++

# bhc-package

Package management for the Basel Haskell Compiler.

## Overview

This crate defines the `bhc.toml` manifest, a lockfile format, a version resolver, a registry client, a Hackage client and a minimal `.cabal` parser. The compiler currently uses only the Hackage part: `bhc --package NAME:VERSION` makes `bhc-driver` call `Hackage::fetch_package`, which downloads and caches the source tarball, reads its `.cabal` file and adds its `hs-source-dirs` to the module search path. The manifest, resolver, registry and `commands` types are not wired into the `bhc` CLI.

Project-level package management (Cabal/Hackage resolution, lockfiles, builds) is handled by hx, which drives bhc as a compiler (see [Build a project with hx](@/docs/quickstart.md#build-a-project-with-hx)). Separately compiled libraries are consumed through `.bhi` interfaces and `--package-db`.

## Modules

| Module | Contents |
|--------|----------|
| `lib.rs` | `Manifest`, `PackageMetadata`, `Edition` (`h26`), `Dependency`, `DetailedDependency`, `PackageId`, `find_manifest` |
| `hackage` | `Hackage`, `HackageConfig`, `HackagePackage` (`source_dirs()`), `PackageVersions` |
| `cabal` | `CabalFile`, `CabalLibrary`, `CabalExecutable`, `CabalDependency`; ignores conditionals and flags |
| `resolve` | `Resolver`, `ResolverConfig`, `Resolution`, `ResolvedDep`, `requirements_compatible` (backtracking, highest compatible version first) |
| `lockfile` | `Lockfile` (`bhc.lock`), `LockedPackage`, `LockedSource`, `find_lockfile`, `compute_checksum` |
| `registry` | `Registry`, `RegistryConfig`, `PackageIndex`, `OfflineRegistry`, publish metadata |
| `commands` | `InitCommand`, `AddCommand`, `RemoveCommand`, `UpdateCommand`, `PublishCommand`, `SearchCommand`, `InfoCommand` |

## Package Manifest

`bhc.toml`, as parsed by `Manifest`:

```toml
[package]
name = "my-app"
version = "1.0.0"
edition = "h26"
profile = "default"

[dependencies]
base = "^1.0"
text = { version = "^2.0", features = ["unicode"] }

[dev-dependencies]
test-framework = "^0.5"
```

Other tables: `[library]`, `[[bin]]`, `[[test]]`, `[[benchmark]]`, `[build-dependencies]` and `[features]`. Versions are semver requirements (`semver::VersionReq`).

## Using Hackage Packages

```bash
# Fetch a package from Hackage and compile against its sources
bhc --package containers:0.6.7 Main.hs
```

Packages are cached locally after the first download.
