+++
title = "bhc-span"
description = "Source location tracking for BHC"
weight = 1
+++

# bhc-span

Source location tracking for the Basel Haskell Compiler.

## Overview

This crate provides the types used to record where things come from in the source, from lexing through code generation. Its only dependency is `serde`. Spans are byte ranges; line and column numbers are computed on demand from a `SourceFile`.

## Key Types

| Type | Description |
|------|-------------|
| `BytePos` | A byte offset into a source file (`u32`) |
| `Span` | Half-open byte range `[lo, hi)`; `Span::DUMMY` for generated code |
| `Spanned<T>` | A value paired with its span (`node`, `span`) |
| `FileId` | Identifier for a source file |
| `FullSpan` | A `Span` together with its `FileId` |
| `LineCol` | 1-indexed line and column |
| `SourceFile` | File name, source text and precomputed line starts |
| `SpanLines` | The lines a span covers (`is_multiline`) |

The source map that holds all files for diagnostics rendering is `bhc_diagnostics::SourceMap`, not part of this crate.

## Usage

```rust
use bhc_span::{BytePos, FileId, SourceFile, Span};

let file = SourceFile::new(FileId::new(0), "Main.hs".into(), src);
let span = Span::new(BytePos::new(0), BytePos::new(9));

// Combine spans
let covering = span1.merge(span2); // min lo, max hi
let through = span1.to(span2);     // span1.lo .. span2.hi

// Source text and position
let text = file.source_text(span);
let pos = file.lookup_line_col(span.lo);
```

Other `Span` methods: `from_raw`, `is_dummy`, `len`, `is_empty`, `shrink_to_lo`, `shrink_to_hi`, `contains`. `SourceFile` also offers `lookup_line`, `num_lines`, `line_content`, `line_start` and `span_lines`.

## Features

- `Span` is two `u32`s, `Copy`, and hashable, so it is cheap to store on every AST and HIR node
- The type checker keys the inferred type of each expression by its `Span`; later phases look types up by span
- Line and column lookup uses binary search over line starts
- The position and span types (`BytePos`, `Span`, `FileId`, `FullSpan`, `LineCol`) implement `Serialize`/`Deserialize`
