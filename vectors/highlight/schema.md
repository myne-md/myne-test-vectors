# `highlight` — dialect parse-recognition

Pins how the Myne markdown dialect parser recognizes `==highlight==` spans.

- **Reference implementation:** `src/dialect/lezer/highlight.ts` (main `myne` repo), composed into `src/dialect/lezer/parser.ts`
- **Spec anchor:** `markdown-dialect.md` §3
- **Runner:** `tests/dialect/dialectParse.test.ts` (`pnpm test`) — TS-side, no Rust generator
- **Generator:** `GEN_DIALECT_VECTORS=1 pnpm vitest run tests/dialect/_genDialectVectors.test.ts`

## What this category pins

The cross-implementation contract is parser-agnostic: **which byte ranges of a
markdown source are recognized as which `highlight` node**. A reference parse tree is
a JS implementation detail; the vectors express ranges as **UTF-8 byte offsets**
so a port that scans bytes (Rust / Swift / Kotlin) can assert against them
directly. The pinned node names are:

- `Highlight`
- `HighlightMark`

This is deterministic cross-client parse behavior — not an on-disk codec and not
cryptographic — but load-bearing: if a port recognizes a different span set, the
same `.myn` content renders differently across clients.

## Case shape

```
cases[]:
  vector_id                 "highlight/v0.1/<slug>"
  case_kind                 "recognize" | "inert"
  input_markdown_utf8_hex   the source, exact UTF-8 bytes (hex)
  informational_markdown    human-readable mirror (runner cross-checks byte-equal)
  expected_dialect_spans    [ { node, from, to } ]  (UTF-8 byte offsets, pre-order)
```

- **`recognize`** — the construct is present; `expected_dialect_spans` lists the
  recognized nodes in document (pre-order) traversal order.
- **`inert`** — a near-miss the parser must NOT recognize; `expected_dialect_spans`
  is empty. These guard against over-recognition (the failure mode a permissive
  port would hit).

## How to consume

Hex-decode `input_markdown_utf8_hex`, run your dialect parser, collect the spans
whose node name is in the set above (converting your parser's offsets to UTF-8
bytes if needed), and assert the ordered list equals `expected_dialect_spans`.
For `inert` cases, assert the list is empty.

## What this does NOT cover

The *rendering* of these constructs (HTML/PDF projection) and their *serialization*
(canonical-D1 emit — see `d1-clip-serializer`) are separate surfaces. This
category pins only recognition: where each construct begins and ends.
