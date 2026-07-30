# `task-inline-format` — dialect task fields: three read dialects, one emitted form

Pins how Myne recognizes, types and re-emits the inline metadata a task line carries.

- **Reference implementation:** `dialect/src/lezer/task-field.ts` (recognition), `dialect/src/taskFields.ts` (typing), `dialect/src/taskFieldWriter.ts` (emission) — main `myne` repo
- **Spec anchor:** `markdown-dialect.md` §3.8
- **Runner:** `tests/dialect/taskVectors.test.ts` (`pnpm test`) — TS-side, no Rust generator
- **Generator:** `GEN_TASK_VECTORS=1 pnpm vitest run tests/dialect/_genTaskVectors.test.ts`

## What this category pins

**Myne reads three dialects and writes one.** Reading recognizes Obsidian's emoji
form (`📅 2026-08-01`), Obsidian's Dataview-style bracketed fields
(`[due:: 2026-08-01]`), and Myne's own key names — so a vault imported from
Obsidian lights up in whichever form the user already had. Writing emits a single
form. Obsidian parity is a **read** goal, not a write goal.

Three layers per case, because agreement on any one of them alone is not enough:

1. **Recognition** — which UTF-8 byte ranges are which node. Offsets are **UTF-8
   byte offsets**, not UTF-16 code-unit indices, so a port that walks bytes
   (Rust / Swift / Kotlin) can assert directly. This is not academic: every emoji
   marker is a surrogate pair in UTF-16 and four bytes in UTF-8, so an
   un-converted vector would disagree on literally every emoji case.
2. **Typing** — the single record all three dialects collapse into. That
   equivalence is what makes the Obsidian importer honest.
3. **Emission** — the whole line as Myne writes it. Editing any field converts
   the **entire** line, which follows from the dialect rule that only directly
   changed lines adopt canonical forms.

## Case shape

```
cases[]:
  vector_id                   "task-inline-format/v0.1/<slug>"
  case_kind                   "dataview" | "emoji" | "myne" | "mixed"
                              | "preserve" | "malformed" | "inert"
  input_markdown_utf8_hex     the source line, exact UTF-8 bytes (hex)
  informational_markdown      human-readable mirror (runner cross-checks byte-equal)
  expected_dialect_spans      [ { node, from, to } ]  (UTF-8 byte offsets, pre-order)
  expected_fields             the typed record; absent fields OMITTED, never null
  expected_emitted_utf8_hex   the whole line as Myne writes it (hex)
  informational_emitted       human-readable mirror
```

- **`dataview` / `emoji` / `myne`** — one of the three read dialects.
- **`mixed`** — both forms on one line, the shape a vault takes mid-migration.
- **`preserve`** — a field Myne recognizes but never types, kept verbatim.
- **`malformed`** — a value that must **not** type and must **not** be corrected.
- **`inert`** — a near-miss the recognizer must not claim; `expected_dialect_spans`
  is empty. These guard over-recognition, the failure mode a permissive port hits.

## Two obligations that are easy to get wrong

**Refusing is not the same as failing.** A value that does not type is
**preserved and surfaced as unparsed** — never discarded, never reported as a
parse error, and never corrected. `2026-02-31` passes the date shape and is still
refused; `2026-08-01T10:00:00Z` carries an offset and therefore does not type at
all, because stripping the offset would silently shift the due date and would put
a coarse geolocation datum into a file that gets exported as plaintext.

**`onCompletion` is preserved and never interpreted.** It is a miniature action
language, and running it would cross the no-code-eval line. Preservation and
non-interpretation are separate obligations and both are normative: its bytes
survive a whole-line rewrite, and it is assigned no meaning whatsoever.

## The fixed-point property

A line already in the emitted form rewrites to **itself**. The runner asserts this
over the whole corpus, not per case, because the consequence is felt at vault
scale: without it, opening a vault would dirty every task line it contains and the
byte-stability guarantee (`markdown-dialect.md` §6 C1) would not hold.

## How to consume

Hex-decode `input_markdown_utf8_hex`, then assert three things: your recognizer's
spans equal `expected_dialect_spans` (converting to UTF-8 byte offsets if your
parser reports otherwise), your typed record equals `expected_fields` (with absent
fields absent rather than null), and your writer's output equals
`expected_emitted_utf8_hex`. For `inert` cases the span list must be empty.

## What this does NOT cover

The recurrence rule **vocabulary** — `repeat` is carried here as an opaque string
and is pinned by the separate `task-recurrence` category. Line-task *restriction*
is also out of scope: the recognizer deliberately recognizes fields on any line,
and applying the `- [ ]` restriction belongs to the consuming layer
(`markdown-dialect.md` §3.8.6).
