# `d1-clip-serializer` vector schema

Pins the **Myne dialect clip serializer** (`serializeClipMarkdown`, `src/dialect/clip.ts` in the [main `myne` repo](https://github.com/myne-md/myne)): the `@lezer/markdown`-tree → canonical-D1 emitter that re-authors the best-effort markdown handed off by the browser extension. Each case is a Defuddle-produced markdown source and the byte-exact canonical-D1 output the serializer emits.

**Reference implementation:** TS-side, in the dialect layer (`src/dialect/clip.ts` + the `clip/` helpers — `clip/math.ts`, `clip/escape.ts`, `clip/captureKind.ts`). There is **no Rust reference implementation** — like `markdown-comment` / `property-coercion`, this is a TS-runner category, not a `protocol`-crate codec.
**Spec:** `docs/spec/markdown-dialect.md` §5 (canonical emit forms) / §6 (round-trip stability). **Ledger:** browser-extension epic `sync-ledger.md` SYNC-7 (a clip is AUTHORED canonical-D1, not a C1-preserved import) + SYNC-11 (canonicalization runs DESKTOP-side via the new tree → canonical-D1 serializer).

## What this category is — and is not

A **deterministic-behavior conformance** category in the `markdown-comment` / `property-coercion` / `note-template` class: not an on-disk codec, not cryptographic — but cross-client behavior every independent implementation (mobile, web) must reproduce **identically**, because *how a clip's markdown canonicalizes* is a divergence risk (two clients that disagree produce different note bytes from the same web page). The contract it guards:

- **A clip is AUTHORED content, not a C1 import.** Unlike the Obsidian-import path (§6, byte-for-byte preserved), a clip may renormalize bytes — canonical delimiters, collapsed whitespace, Setext→ATX, reference-link→inline, indented-code→fenced — as long as the emit re-parses to a structurally-identical tree. That round-trip invariant (`emit → parser.parse(emit) → same tree shape`) plus idempotency (`serialize(serialize(x)) === serialize(x)`) is the checkable definition of "canonical D1". Every output pinned here is a serializer fixed point.
- **The fidelity floor (`floor` / `math`).** Faithful **tables** (canonical single-space-padded GFM, per-column alignment + pipe-escaping preserved), **code** (fenced with verbatim body + language hint, degrade-safe fence length), and **math** (`$…$` / `$$…$$` emitted tokenizer-legal against D1's `$`-disambiguation). The hard floor: a clip that reads like an authored note keeps these intact.
- **The article wedge (`wedge`).** Headings (ATX), lists (bullet `- ` / ordered `1. `, nested), inline links `[text](url)`, the emphasis family (`*`/`**`/`~~`), callouts (`> [!type]`), task lists (`- [x]`/`- [ ]`, lowercase x), footnotes (`[^id]` + `[^id]:`), and the default highlight (`==text==`).
- **The normalize cases (`normalize`).** Recognized-on-parse → canonical emit: Setext underline → ATX, reference-style link + `[ref]: url` definition → one inline link, non-canonical `_italic_`/`__bold__` → `*`/`**`, indented code block → fenced. (A C1 import would preserve these byte-for-byte; a clip re-authors them.)
- **The safe-degrade cases (`degrade`).** A construct with **no Myne emit form** is dropped — never emitted as broken markdown. An inline/block `%%comment%%` full-drops (§6 lossy-strip); sub/superscript drop their markup but **keep** the wrapped text (`H~2~O` → `H2O`). The serializer never throws and never emits a malformed span.

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/d1-clip-serializer.json`](../../schema/v0.1/d1-clip-serializer.json) (per-case shape). Every case carries a markdown source as `input_markdown_utf8_hex` (+ an `informational_markdown` mirror) and pins the canonical-D1 output. One case shape, discriminated by `case_kind`:

- `case_kind` — `floor` (faithful table / code / inline-code), `math` (`$…$` / `$$…$$` against the D1 `$`-tokenizer, incl. the degrade-to-inert-text path — task 12 extends this subset), `wedge` (the article constructs), `normalize` (recognized → canonical), or `degrade` (a no-emit-form construct dropped).
- `expected_canonical_d1_utf8_hex` — the serializer output as exact UTF-8 bytes (hex): what `serializeClipMarkdown(input)` emits, byte-for-byte.
- `informational_canonical_d1` — optional human-readable mirror of the output; cross-checked byte-equal against the hex.

## How to consume

A port runs each case:

1. Hex-decode `input_markdown_utf8_hex` to the markdown source.
2. Run the port's clip serializer (its `@lezer/markdown`-tree → canonical-D1 equivalent).
3. Assert the output **bytes** equal `expected_canonical_d1_utf8_hex`.

Compare bytes, not rendered strings. (The round-trip + idempotency invariants are pinned by the main repo's per-construct tests; this category pins the exact emitted bytes.)

## Cases (23)

5 `floor` + 3 `math` + 9 `wedge` + 4 `normalize` + 2 `degrade`. The load-bearing pins:

| `vector_id` | pins |
|---|---|
| `floor-table-aligned` | a GFM table → canonical single-space-padded form, per-column alignment preserved |
| `floor-table-pipe-escape` | a literal `\|` in a cell stays escaped so the table re-parses to the same columns |
| `floor-fenced-code-lang` | a fenced block keeps its language hint + verbatim body bytes |
| `floor-fenced-code-backtick-run` | inner ```` ``` ```` run → outer fence is one longer (re-parses as one block) |
| `floor-inline-code` | inline code → canonical backtick form |
| `floor-block-math` | a standalone `$$…$$` → canonical multi-line block |
| `floor-inline-math` | tokenizer-legal `$…$` rides verbatim |
| `wedge-callout-note` | `> [!NOTE]` → canonical lower-cased `> [!note]` callout |
| `wedge-task-list` | `- [x]` / `- [ ]` (lowercase x) |
| `normalize-setext-to-atx` | Setext underline → ATX (`# `/`## `) |
| `normalize-ref-link-to-inline` | `[t][ref]` + `[ref]: url` → one inline `[t](url)` |
| `normalize-alt-emphasis-delims` | `_italic_`/`__bold__` → `*`/`**` |
| `normalize-indented-code-to-fenced` | indented code block → fenced |
| `degrade-drop-comment` | inline `%%…%%` → full-dropped |
| `degrade-drop-subsup-keep-text` | sub/superscript markup dropped, text kept (`H~2~O` → `H2O`) |
| `degrade-math-illegal-to-inert` | `$ 5` / `$10` never emit as an opening `$…$` span (currency guard) |

`normalize-setext-to-atx` and `normalize-ref-link-to-inline` are the most divergence-prone — they fail any implementation that ports the C1 byte-preserve rule instead of the clip-is-authored re-author rule.

## Invariants the vectors pin

- **Authored, not preserved.** A clip re-authors recognized-on-parse non-canonical forms to canonical emit; it does NOT byte-preserve them (that is the C1 import rule, §6).
- **Round-trip + idempotent.** Every pinned output re-parses to a structurally-identical tree and is a serializer fixed point (`serialize(serialize(x)) === serialize(x)`).
- **Fidelity floor intact.** Tables / code / math survive faithfully; math is tokenizer-legal against D1's `$`-disambiguation or degrades to inert text.
- **Safe-degrade, never broken markdown.** A no-emit-form construct is dropped; the serializer never throws and never emits a malformed span.

## What this does NOT cover

- **Frontmatter / provenance** — `source_url` / `clip_date` / `folder` / `tags` ride the handoff envelope and are composed into frontmatter by desktop-ingest (sub-feature 3), not emitted by the serializer.
- **`[[wikilink]]` resolution + attachment import** — desktop-ingest (sub-feature 3, SYNC-11). The serializer emits only the `clip-image:N` reference seam (task 9), never an `att:<id>` or a fetched byte.
- **The `capture_kind` discriminator + no-content outcome** — full vs highlight route through the same `serializeClipMarkdown` (task 8); the no-content signal is structural (task 10). Both are main-repo behaviors, not byte-vectored here.

## Regeneration

Hand-curated against §5/§6 — **no Rust generator** (the serializer lives TS-side). The cases were emitted from the live reference (`serializeClipMarkdown`) by a one-off main-repo Vitest helper that hex-encodes each realistic Defuddle-style fixture and records the live serializer output (every output was additionally confirmed to be a serializer fixed point). Conformance runs **TS-side**: the dialect Vitest runner (`tests/dialect/d1ClipSerializer.test.ts`) hex-decodes each case, runs the live `serializeClipMarkdown`, and asserts the output bytes. The dedicated `$`-tokenizer math subset is extended by capture-extraction task 12. A failing vector in any port (or the desktop TS impl) is a stop-the-world bug — read why before regenerating.
