+++
title = "bhc-lexer"
description = "Tokenization for bhc"
weight = 10
+++

# bhc-lexer

Lexical analysis for the Basel Haskell Compiler.

## Overview

This crate turns Haskell source into a stream of `Spanned<Token>` values. It is a hand-written lexer (it does not use `logos`, although the crate lists it as a dependency) covering Haskell 2010 tokens plus the GHC and H26 extensions the parser accepts: Unicode identifiers and operators, qualified names, documentation comments, pragmas, MagicHash literals and the layout rule.

## Key Types

| Type | Description |
|------|-------------|
| `Lexer<'src>` | The tokenizer; implements `Iterator<Item = Spanned<Token>>` |
| `LexerConfig` | Doc-comment and pragma preservation, tab warnings, tab width (default 8) |
| `Token` | A token, wrapping `kind: TokenKind` |
| `TokenKind` | Token enumeration |
| `IntLiteral`, `FloatLiteral`, `NumBase`, `NumSuffix` | Numeric literal payloads (base, MagicHash suffix) |
| `LexError` | Lexical error carried in `TokenKind::Error` |

## Token Categories

| Category | Examples |
|----------|----------|
| Keywords | `let`, `where`, `case`, `of`, `do`, `mdo`, `rec`, `forall`, `hiding` |
| Names | `Ident`, `ConId`, `QualIdent`, `QualConId` (`Data.List.map`) |
| Operators | `Operator`, `ConOperator` (`:+`), qualified forms, `->`/`→`, `=>`/`⇒`, `::`/`∷`, `@`, `~`, `!`, `#` |
| Literals | `IntLit`, `FloatLit`, `CharLit`, `StringLit`, `RawStringLit` (H26 `r#"..."#`) |
| Brackets | `(#`/`#)` (unboxed tuples), `[|`/`|]` (quasiquotes), `'[` (promoted lists), `''` (TH name quote) |
| Layout | `VirtualLBrace`, `VirtualSemi`, `VirtualRBrace` |
| Trivia kept as tokens | `DocCommentLine`, `DocCommentBlock`, `Pragma` |

## Usage

```rust
use bhc_lexer::{lex, Lexer, LexerConfig, TokenKind};

let source = "let x = 42 in x + 1";

for tok in Lexer::new(source) {
    println!("{:?} at {:?}", tok.node.kind, tok.span);
}

// Or collect everything at once:
let tokens = lex(source);
let tokens = bhc_lexer::lex_with_config(source, LexerConfig::default());
assert!(matches!(tokens.last().unwrap().node.kind, TokenKind::Eof));
```

## Layout Handling

After `where`, `let`, `do` or `of`, the lexer inserts virtual braces and semicolons based on indentation, so the parser sees explicit block structure:

```haskell
-- Source
do
  x <- getLine
  print x

-- Tokens include virtual { ; }
do { x <- getLine ; print x }
```
