+++
title = "bhc-interface"
description = "Module interfaces"
weight = 64
+++

# bhc-interface

Module interface files for the Basel Haskell Compiler.

## Overview

This crate defines the `.bhi` interface format: the public API of a compiled module, used to type check and lower importers without their source. `bhc -c` writes one `.bhi` per module (into `--hidir`), and importers find them through the import paths or a `--package-db` directory.

Interfaces carry types, classes and instances but not function bodies. Core bodies for cross-module specialization travel separately in the `.bhc` sidecar that `bhc-driver` writes next to each `.bhi`.

## Modules

| Module | Contents |
|--------|----------|
| crate root | `ModuleInterface` and its parts; reading/writing; `interface_path` |
| `generate` | `generate_interface(module_name, &ast, &typed, ...)` builds a `ModuleInterface` from the AST and the type checker's `TypedModule` |
| `convert` | `TypeConverter`: interface `Type`/`TypeSignature`/`Kind` to `bhc_types::Ty`/`Scheme`/`Kind`, plus constructor schemes |

## Interface Contents

`ModuleInterface` fields:

| Field | Description |
|-------|-------------|
| `header` | `InterfaceHeader`: format version, module name, module hash, compiler version |
| `values` | `ExportedValue`: name, `TypeSignature`, `InlineInfo`, optional arity |
| `types` | `ExportedType`: params, `Kind`, optional `TypeDefinition` (constructors with field names) |
| `classes` | `ExportedClass`: params, superclasses, methods, functional dependencies |
| `instances` | `ExportedInstance`: class, types, context, method names |
| `instance_methods`, `class_defaults` | Symbol names and arities of instance methods and class default methods |
| `pattern_synonyms` | `ExportedPatternSynonym` |
| `reexports` | Re-exported names mapped to their defining module |
| `dependencies` | `InterfaceDependency` list |

## File Format

```
magic:   "BHCI"            (INTERFACE_MAGIC)
payload: bincode-encoded ModuleInterface
         header.version must equal INTERFACE_VERSION (currently 5)
```

`interface_path(dir, "Data.List")` returns `dir/Data/List.bhi`.

## Usage

```rust
use bhc_interface::{interface_path, ModuleInterface};

let iface = bhc_interface::generate::generate_interface(
    "Data.Foo", &ast, &typed, &imported_class_params, &imported_class_methods,
);
iface.write_to_file(interface_path(&hidir, "Data.Foo"))?;

let loaded = ModuleInterface::read_from_file("hi/Data/Foo.bhi")?;
if let Some(v) = loaded.lookup_value("foo") { /* v.signature */ }
```

A version or magic mismatch is reported as `InterfaceError::VersionMismatch` or `InvalidFormat`.

## Benefits

- Separate compilation: importers do not re-parse dependencies
- Package databases of precompiled modules (`--package-db`)
- Instances and class methods resolve across module boundaries
