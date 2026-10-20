+++
title = "bhc-diagnostics"
description = "Error reporting for bhc"
weight = 6
+++

# bhc-diagnostics

Error and warning reporting for the Basel Haskell Compiler.

## Overview

This crate defines the structured `Diagnostic` type shared by every compiler phase, a `SourceMap` of loaded files, and several output formats: terminal text, JSON, and LSP protocol types. It also holds the long-form error explanations keyed by code.

## Key Types

| Type | Description |
|------|-------------|
| `Diagnostic` | A message with severity, optional code, labels, notes and suggestions |
| `Severity` | `Bug`, `Error`, `Warning`, `Note`, `Help` |
| `Label` | A `FullSpan` plus message, primary or secondary |
| `Suggestion` | Message, span, replacement text and `Applicability` |
| `Applicability` | `MachineApplicable`, `MaybeIncorrect`, `HasPlaceholders`, `Unspecified` |
| `DiagnosticHandler` | Collects diagnostics and counts errors/warnings |
| `SourceMap` | Files registered by name, looked up by `FileId` |
| `DiagnosticRenderer` | Compact terminal renderer (used by the driver today) |
| `CargoRenderer` / `RenderConfig` | Cargo-style renderer with multi-line snippets (`render` module) |
| `IntoDiagnostic` | Trait for converting phase-specific errors into a `Diagnostic` |

## Modules

| Module | Contents |
|--------|----------|
| `render` | `CargoRenderer`, `RenderConfig::{colored, plain}`, ANSI `colors` |
| `json` | `JsonDiagnostic` and `diagnostic_to_json`, `to_json_string`, `to_json_lines` |
| `lsp` | LSP types (`LspDiagnostic`, `LspCodeAction`, `LspHover`, ...) and `to_lsp_diagnostic`, `to_code_actions`, `publish_diagnostics` |
| `explain` | `ErrorExplanation` table; `get_explanation`, `all_error_codes`, `print_explanation` |

## Usage

```rust
use bhc_diagnostics::{
    Applicability, Diagnostic, DiagnosticHandler, DiagnosticRenderer, SourceMap, Suggestion,
};

let mut sm = SourceMap::new();
let file = sm.add_file("Main.hs".into(), src);

let diag = Diagnostic::error("type mismatch")      // also ::warning, ::bug
    .with_code("E0001")
    .with_label(span, "expected `Int`, found `String`")   // span: FullSpan
    .with_note("cannot unify Int with String")
    .with_suggestion(Suggestion::new(
        "use `show` to convert",
        span,
        "show x",
        Applicability::MaybeIncorrect,
    ));

let mut handler = DiagnosticHandler::new();
handler.emit(diag);

DiagnosticRenderer::new(&sm).render_all(handler.diagnostics());
```

## Output Format

`DiagnosticRenderer` prints a header, a location line per label, the spanned source text, then notes and suggestions:

```
error[E0001]: type mismatch
 --> Main.hs:10:5
   |
   | x + "hello"
   | ^^^^^^^^^^^
   | expected `Int`, found `String`
 = note: cannot unify Int with String
 = help: use `show` to convert
   |
   | show x
```

`CargoRenderer` produces the fuller Cargo-style layout with line-number gutters and multi-line spans.

## Error Codes

`explain` contains explanations (title, description, common mistakes, related codes) for type-error codes such as `E0001` (type mismatch) and `E0002` (infinite type). Codes are carried as strings on `Diagnostic::code`.
