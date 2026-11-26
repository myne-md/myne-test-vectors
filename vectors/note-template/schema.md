# `note-template` vector schema

Pins `protocol::template::render` + `protocol::template::supported_tokens` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)) — the deterministic **note-template variable engine** (MYND-188): `{{token}}` substitution against a closed, case-sensitive 7-token registry.

**Reference implementation:** `protocol/src/template.rs::{render, supported_tokens, RenderContext}`.
**Contract:** `design notes` 2026-06-07 "Note templates" (the deterministic-substitution / no-scripting commitment).

## What this category is — and is not

This is a **deterministic-behavior conformance** category in the `password-policy` class: not an on-disk codec, not cryptographic — but cross-client behavior every independent implementation (mobile, web) must reproduce **bit-identically**. `render(template, ctx)` is a pure function: one left-to-right pass, no clock, no locale, no I/O. The cross-implementation risks it guards are specific:

- **Single pass, never re-scan.** A substituted value containing `{{date}}` stays literal in the output. An implementation that re-scans its output (or substitutes values before scanning) produces different bytes — and opens an injection-shaped behavior the engine deliberately does not have.
- **Verbatim passthrough IS the escape mechanism.** Unknown tokens (`{{project}}`), format-arg shapes (`{{title:foo}}`), uppercase names (`{{Date}}`) and stray/unbalanced braces all pass through byte-identical. There is no `\{{` escape syntax and no format mini-language — the anti-plugin commitment (`vision.md`) expressed in bytes.
- **No unicode normalization.** The engine is byte-transparent: NFD combining sequences stay NFD. The hex pins are the contract; JSON-string mirrors could be silently NFC-normalized by tooling, which is exactly why the load-bearing fields are `*_utf8_hex`.
- **Host-injected time.** `ctx.date`/`time`/`datetime`/`weekday` arrive pre-formatted in **local** time from the host; the engine performs no calendar arithmetic and no UTC conversion (the shared fixture is deliberately a 01:00-local instant that is the *previous* day in UTC).

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/note-template.json`](../../schema/v0.1/note-template.json) (per-case shape). Two case kinds:

`case_kind: "render"`:

- `vector_id` — stable `note-template/v0.1/<slug>` identifier.
- `input_template_utf8_hex` — the template body as exact UTF-8 bytes (hex; empty allowed).
- `input_ctx_{title,date,time,datetime,weekday,period_start,period_end}_utf8_hex` — the seven `RenderContext` fields as exact UTF-8 bytes (title may be empty). `period_start`/`period_end` are defined for EVERY period, daily included, so the two tokens never render empty.
- `expected_output_utf8_hex` — the exact bytes `render` MUST produce.
- `informational_template_utf8` / `informational_output_utf8` — optional human-readable mirrors; the reference runner cross-checks them byte-equal against the hex.

`case_kind: "token_registry"`:

- `expected_tokens` — the exact, **order-sensitive** return of `supported_tokens()` (`date`, `time`, `title`, `datetime`, `weekday`, `period_start`, `period_end` — the two period tokens are APPENDED, so the five v1 names keep their positions and the v1 order stays a prefix of this one).

## How to consume

A port runs each `render` case:

1. Hex-decode the template and the seven ctx fields; interpret as UTF-8.
2. Invoke the port's `render` equivalent with that context.
3. Assert the output **bytes** equal `expected_output_utf8_hex` — compare bytes, not parsed/normalized strings.

For `token_registry`: assert the port's token list equals `expected_tokens` element-wise in order. The set must also be exactly what the port's renderer substitutes — a port advertising a token it does not substitute (or vice versa) has split the single registry this category exists to pin.

## Cases (13)

Shared fixture context unless noted: title `My Note`, date `2026-06-07`, time `01:00`, datetime `2026-06-07 01:00`, weekday `Sunday`.

| `vector_id` | pins |
|---|---|
| `render-all-five-tokens` | all 5 tokens in one pass; local-day (no UTC off-by-one) |
| `render-repeated-token-multiline` | every occurrence substitutes; `\n` byte-transparent |
| `render-unknown-token-verbatim` | unknown token verbatim between substituting neighbors |
| `render-case-sensitive-uppercase-literal` | `{{Date}}` / `{{DATE}}` are not tokens |
| `render-format-arg-literal` | `{{title:foo}}` literal — no mini-language |
| `render-stray-unbalanced-braces` | `{{ }}`, lone `}}`, unclosed `{{`, single braces all verbatim |
| `render-open-brace-before-token` | `{{ {{date}}` → `{{ 2026-06-07` (rescan after literal `{{`) |
| `render-empty-template` | `""` → `""` |
| `render-empty-title` | empty value substitutes as empty string (SYNC-5-R) |
| `render-title-not-rescanned` | **the single-pass pin** — `{{date}}` inside the injected title stays literal |
| `render-unicode-weekday-and-combining` | NFD + Turkish weekday byte-transparency, no normalization |
| `token-registry-order` | `supported_tokens()` order + set |

`render-title-not-rescanned` is the load-bearing case: it is the one vector that fails on any re-scanning implementation while every simpler case still passes.

## Invariants the vectors pin

- **Determinism.** Output is a pure function of `(template, ctx)` — byte-identical across calls, machines, and implementations.
- **Closed vocabulary, case-sensitive.** Exactly seven lowercase tokens substitute; everything else `{{…}}`-shaped passes through verbatim.
- **Single eager pass.** Substituted values are never re-scanned; no escaping is applied to injected values (plain-markdown output).
- **Byte transparency.** Multi-byte UTF-8 and NFD combining sequences survive untouched in both template text and injected values.
- **Registry/accessor unity.** `supported_tokens()` and `render` share one registry — set and order pinned together.

## What this does NOT cover

- **Host formatting policy.** How the host formats `date`/`time`/`datetime`/`weekday` (locale, format strings, daily-notes `daily_date_format`) is desktop-layer behavior, pinned by its own tests — the engine just substitutes the bytes it is given.
- **Empty-title UX.** Whether an empty title prompts the user or renders empty is the new-note-from-template flow's decision (SYNC-5-R); the engine only pins that an empty value substitutes as an empty string.
- **Future tokens.** Adding a registry token is an additive v0.1.x change: new vectors land alongside, existing `vector_id`s stay immutable, and the `token_registry` case is regenerated in the same change.

## Regeneration

```bash
cargo test -p protocol --lib emit_note_template_vectors_for_paste -- --ignored
cp /tmp/myne_note_template_vectors.json vectors/note-template/v0.1/cases.json
```

The generator lives in `protocol/src/template.rs::tests::emit_note_template_vectors_for_paste`. `#[ignore]`-gated; emits ready-to-copy JSON and self-asserts every hand-written expectation against the live engine before writing. Conformance against the live Rust impl runs via `cargo test -p protocol --test vectors note_template_vectors`.
