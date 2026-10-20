+++
title = "bhc-parser"
description = "Parsing for BHC"
weight = 12
+++

# bhc-parser

Parser for the Basel Haskell Compiler.

## Overview

This crate parses Haskell source code into a `bhc_ast::Module`. It is a hand-written recursive descent parser with precedence climbing for infix expressions. `Parser::new` runs the lexer (`bhc-lexer`, which applies the layout rule) and parses the resulting token stream. CPP is handled before parsing, by the preprocessor in `bhc-driver`.

Dependencies: `bhc-span`, `bhc-intern`, `bhc-lexer`, `bhc-ast`, `bhc-diagnostics`.

## Key Types

| Type | Description |
|------|-------------|
| `Parser<'src>` | Token stream, position, diagnostics, recursion depth, declared fixities |
| `ParseError` | `Unexpected`, `UnexpectedEof`, `InvalidLiteral`, `RecursionLimit`; `to_diagnostic(file)` |
| `ParseResult<T>` | `Result<T, ParseError>` |

The parser is split into `decl` (module header, imports, declarations), `expr`, `pattern` and `types`.

## Usage

```rust
use bhc_parser::{parse_module, parse_expr};

// Each returns the result (if any) plus all diagnostics collected
let (module, diags) = parse_module(source, file_id);
let (expr, diags) = parse_expr(source, file_id);
```

`parse_import_decl` parses a single `import` line; the REPL uses it.

## Error Recovery

When a top-level declaration fails to parse, the parser records a diagnostic and skips to the next declaration (`recover_to_next_decl`), so one run can report several errors. Nesting of expressions, types and patterns is capped at a depth of 64 (`RecursionLimit`) so that deeply nested input produces a diagnostic instead of a stack overflow.

## Template Haskell Syntax

A prefix `$(...)` in expression position parses as an application of the builtin `$splice`. Declaration splices become `Decl::Splice` and are expanded later by [bhc-lower](@/docs/internals/compiler/bhc-lower.md). Name quotes `''T` and `'x` parse as ordinary constructor and variable references.

## Operator Precedence

Infix expressions use precedence climbing. Fixities come from two sources:

- Fixity declarations in the module being parsed (`infixl 1 \`op\``, `infix 0 ~~>`). A pre-scan of the token stream collects them, so they apply wherever they appear relative to the operator's uses.
- A built-in table for Prelude and common library operators (`.`, `$`, `>>=`, `<$>`, `<|>`, `:|`, Arrow combinators). Anything else defaults to `infixl 9`.

Fixity declarations are not propagated across modules yet.

```haskell
1 + 2 * 3      -- 1 + (2 * 3)
a >>= b >>= c  -- (a >>= b) >>= c
```
