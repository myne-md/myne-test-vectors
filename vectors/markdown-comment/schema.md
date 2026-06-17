# `markdown-comment` vector schema

Pins the **Myne dialect `%%…%%` editor-only comment** (`docs/spec/markdown-dialect.md` §3.1 block / §3.2 inline / §3.7.5 exclusion zones / §6 strip, MYND-315, in the [main `myne` repo](https://github.com/myne-md/myne)): which `%%` runs a conformant client recognizes as a comment, and what it strips on a lossy projection.

**Reference implementation:** TS-side, in the dialect layer (`src/dialect/lezer/comment.ts` parser config + `src/dialect/stripComments.ts` strip primitive). There is **no Rust reference implementation** — like `property-coercion`, this is a TS-runner category, not a `protocol`-crate codec.
**Contract:** `design notes.md` 2026-06-17 "Comments enter the dialect: `%%…%%`".

## What this category is — and is not

A **deterministic-behavior conformance** category in the `property-coercion` / `note-template` class: not an on-disk codec, not cryptographic — but cross-client behavior every independent implementation (mobile, web) must reproduce **identically**, because *which `%%` spans a client hides and strips on export* is a divergence risk. The risks it guards:

- **Terminated-only recognition.** A comment requires BOTH delimiters: terminated inline `%%…%%` and terminated block `%%`…`%%` (same line or multi-line) are comments; an **unterminated `%%` is INERT** — no node, stays literal text, hides and strips nothing. This is a **deliberate divergence from Obsidian**, whose unterminated `%%` swallows the rest of the document into a comment. The inert rule is the safe choice (no "half my note vanished" surprise); it is the load-bearing cross-client tie-break (`inline-unterminated-inert`, `block-unterminated-inert`).
- **Structural exclusion zones (§3.7.5).** A `%%` inside a fenced code block, inline code, a wikilink interior, or an inline-math leaf is **literal**, not a comment (`exclusion-*` cases). A conformant parser inherits this structurally (the `%%` recognizer runs only at outer-inline-text level / on a paragraph-class line).
- **The strip output.** The canonical "strip `%%` comments" primitive removes each recognized comment node's **byte range** from the source, leaving every other byte verbatim. This is the single output every lossy projection (HTML / PDF / stripped-TXT / document statistics) shares; it is NOT applied to the byte-verbatim `.md` export (SYNC-2 — comments are author content that survive a walk-away).
- **Single `%` is never a comment** (`single-percent-not-comment`): `100% done` stays text.

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/markdown-comment.json`](../../schema/v0.1/markdown-comment.json) (per-case shape). Every case carries a markdown source as `input_markdown_utf8_hex` (+ an `informational_markdown` mirror) and pins recognition + the strip output. One case shape, discriminated by `case_kind`:

- `case_kind` — `recognize` (the source contains a comment that strips), `inert` (an unterminated `%%` / single `%` — no comment, no strip), or `exclusion` (`%%` inside a §3.7.5 zone — no comment, no strip).
- `expected_has_comment` — boolean: whether the source contains ≥1 recognized `%%…%%` comment node. `true` only for `recognize`; `false` for `inert` / `exclusion`.
- `expected_stripped_utf8_hex` — the canonical strip output as exact UTF-8 bytes (hex): the source with every recognized comment span's byte range removed, all other bytes verbatim. For `inert` / `exclusion` cases this equals `input_markdown_utf8_hex`.
- `informational_stripped` — optional human-readable mirror of the strip output; cross-checked byte-equal against the hex.

## How to consume

A port runs each case:

1. Hex-decode `input_markdown_utf8_hex` to the markdown source.
2. Parse it with the port's dialect parser and ask: does it contain a `%%…%%` comment node? Assert equals `expected_has_comment`.
3. Run the port's canonical comment-strip (remove each recognized comment node's byte range). Assert the output **bytes** equal `expected_stripped_utf8_hex`.

Compare bytes, not rendered strings.

## Cases (14)

The load-bearing tie-breaks:

| `vector_id` | pins |
|---|---|
| `inline-basic` | `before %%a note%% after` → strips to `before  after` |
| `inline-empty` | `%%%%` is a valid empty comment, strips to nothing |
| `inline-unterminated-inert` | a lone `%%` with no closer → inert (no strip) |
| `single-percent-not-comment` | `100% done` → not a comment |
| `block-single-line` | `%% a whole-line note %%` → strips |
| `block-multiline` | a multi-line `%%`…`%%` block strips entirely |
| `block-unterminated-inert` | an unterminated block `%%` → inert; prose below survives |
| `exclusion-fenced-code` | `%%` inside ``` fences → literal |
| `exclusion-inline-code` | `%%` inside `` `…` `` → literal |
| `exclusion-wikilink` | `%%` inside `[[…]]` → literal |
| `exclusion-inline-math` | `%%` inside `$…$` → literal |

`block-unterminated-inert` and `inline-unterminated-inert` are the most divergence-prone — they fail any implementation that ports Obsidian's EOF-swallow instead of the Myne inert rule.

## Invariants the vectors pin

- **Both delimiters required.** Terminated `%%…%%` (inline + block) is a comment; an unterminated `%%` is inert.
- **Structural exclusion** of code / inline-code / wikilink / math-leaf zones.
- **Strip = byte-range removal** of recognized comment nodes; all other bytes verbatim.
- **`.md` is exempt** — only lossy projections strip (asserted in the main repo, not vectored here, because the `.md` path runs no strip by construction).

## What this does NOT cover

- **Editor decoration / cursor-aware reveal** — a desktop-render concern, not a cross-client byte contract.
- **The downstream lossy projection's OTHER transforms** (heading/emphasis stripping, etc.) — those belong to the projection, not the comment rule; this category pins only the comment-removal delta.
- **The `.md` byte-verbatim export** — it runs no strip (SYNC-2); asserted in the main repo's export-flow tests.

## Regeneration

Hand-curated against §3.1/§3.2/§3.7.5/§6 — **no Rust generator** (the recognizer + strip live TS-side). The cases were emitted from the live reference (`stripDialectComments` / `hasDialectComment`) by a one-off main-repo Vitest helper that hex-encodes each source and records the live recognition + strip; conformance runs **TS-side** (the dialect Vitest runner hex-decodes each case, runs the live recognizer + strip, and asserts recognition + the strip bytes). A failing vector in any port (or the desktop TS impl) is a stop-the-world bug — read why before regenerating.
