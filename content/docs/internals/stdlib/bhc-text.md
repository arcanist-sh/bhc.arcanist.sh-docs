+++
title = "bhc-text"
description = "Text processing"
weight = 5
+++

# bhc-text

Text and byte-string support for the Basel Haskell Compiler.

## Overview

This crate implements `Data.Text`, `Data.Text.Lazy`, `Data.Text.Encoding`, `Data.Text.IO`, `Data.ByteString`, `Data.ByteString.Lazy`, `Data.ByteString.Builder` and related modules in Rust, as about 170 C functions. It is built as `libbhc_text.a`, which the native backend links into every program, and the compiler lowers `Data.Text` and `Data.ByteString` operations to calls into it. The `hs/BHC/Data/Text*.hs` and `hs/BHC/Data/ByteString*.hs` sources are not compiled into programs.

## Representation

| Type | Layout |
|------|--------|
| `Text` | 24-byte header `[data_ptr][offset][byte_len]` followed by UTF-8 bytes |
| `ByteString` | Same header layout as `Text`, raw bytes |
| Lazy `Text` | Cons list of strict `Text` chunks: `Empty` (tag 0) or `Chunk` (tag 1, chunk, rest), as in GHC |
| Lazy `ByteString` | Cons list of strict `ByteString` chunks |
| `ByteString.Builder` | Chunk list of strict `ByteString`s, the same layout as a lazy `ByteString` |

Because `data_ptr` may point into another value's buffer, `take`, `drop` and similar operations create a new header over the same bytes instead of copying them. They still walk the UTF-8 data to find the byte offset of the n-th character, so they are O(n) in characters. The shared layout also lets `Data.Text.Encoding` convert between `Text` and `ByteString` by sharing the buffer.

With `{-# LANGUAGE OverloadedStrings #-}`, a string literal used at type `Text` is built as a `Text` value.

## Rust modules

| Module | Contents |
|--------|----------|
| `text` | Strict `Text`: construction, slicing, search, case conversion, splitting, folding |
| `lazy_text` | Lazy `Text` chunks, `fromChunks`, `toStrict`, ... |
| `text_io` | `Data.Text.IO` file and handle functions working directly on `Text` |
| `bytestring` | Strict `ByteString` |
| `lazy_bytestring` | Lazy `ByteString`, lazy `Char8` and lazy `Text` encoding |
| `builder` | `ByteString.Builder` constructors (`stringUtf8`, `charUtf8`, `intDec`, hex and binary encodings) |
| `bytearray` | Mutable byte buffers used by `ByteString` |
| `simd` | `bhc_utf8_validate`, `bhc_memchr`, substring search, ASCII case conversion |

Despite the module name, the `simd` functions are scalar Rust loops today (their source carries TODOs for SIMD versions); they are correct, not vectorized.

## What works today

Checked against GHC 9.8.2 on 2026-10-20, one expression per program:

- `Data.Text`: `pack`, `unpack`, `empty`, `singleton`, `length` (counts code points), `null`, `take`, `takeWhile`, `reverse`, `map`, `filter`, `toUpper`, `words`, `lines`, `splitOn`, `intercalate`, `concat`, `isPrefixOf`, `strip`, `replace`, `count`, `show`
- `Data.Text.IO`: `putStrLn`, `readFile`, `writeFile`
- `Data.Text.Encoding`: `encodeUtf8`, `decodeUtf8`
- `Data.Text.Lazy`: `pack`, `fromChunks`, `length`, `toStrict`
- `Data.ByteString`: `pack`, `unpack`, `length`
- `Data.ByteString.Builder`: `stringUtf8`, `toLazyByteString`

## Known gaps

- `Data.Text.splitAt`, `breakOn`, `zip`, `justifyLeft`, `Data.Text.Encoding.decodeUtf8'` and `decodeLatin1`, and `Data.Text.Lazy.Builder.toLazyText` are stubs that abort when run.
- `Data.ByteString.Char8.pack` fails to compile.
- `BB.hPutBuilder stdout (... <> BB.intDec 42 <> ...)` fails to type-check.
- `show` of non-ASCII `Text` uses Rust escapes: `"\u{e0}b"` where GHC prints `"\224b"`.
