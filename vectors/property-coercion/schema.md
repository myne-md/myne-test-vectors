# `property-coercion` vector schema

Pins the **Saved Views property value-typing / coercion ruleset** (`docs/spec/markdown-dialect.md` §3.6.4, MYND-271, in the [main `myne` repo](https://github.com/myne-md/myne)): how a YAML frontmatter value coerces to exactly one of five Myne property types — `string` / `number` / `date` / `list` / `checkbox`.

**Reference implementation:** TS-side, in the property-index sub-feature (`src/notes/` — Build-B: TS parses + types, Rust stores opaque pre-typed values). There is **no Rust reference implementation** — this is the first category whose reference impl is TypeScript, not the `protocol` crate.
**Contract:** `design notes.md` 2026-06-14 "Saved Views property-index: pull the deferred §3.6/§8 YAML-body parse forward + full value typing".

## What this category is — and is not

A **deterministic-behavior conformance** category in the `password-policy` / `note-template` class: not an on-disk codec, not cryptographic — but cross-client behavior every independent implementation (mobile, web) must reproduce **identically**. Coercion is a pure function of one YAML value; property-index is the **single type authority** for the Saved Views epic, and every sibling feature (query engine, view model, render, `.base` importer) consumes this typing and never redefines it. The cross-implementation risks it guards:

- **YAML 1.2 core schema, not 1.1.** Booleans are exactly `true | True | TRUE | false | False | FALSE` (core schema §10.2.1) — `yes` / `no` / `on` / `off` are **strings** (the 1.1 "Norway problem" does not occur). Core ints permit leading zeros and hex/octal (`007` → 7, `0x1A` → 26). An implementation on a YAML 1.1 parser (or the stricter JSON schema) diverges on these. The reference resolver is `js-yaml` `CORE_SCHEMA`.
- **The quoting asymmetry.** Quoting forces a YAML string node, demoting a value **out of** `checkbox` and `number` (`true` / `True` → checkbox, `"true"` → string; `1.0` → number, `"1.0"` → string) — but **not out of `date`**, because the date recognizer runs on the resolved string (YAML has no native timestamp in core, and erases the plain-vs-quoted distinction for strings).
- **Number normalization = canonical decimal; date = verbatim.** A `number`'s normalized form is the canonical decimal of the parsed IEEE-754 binary64 value (`1.0` → `1`, `007` → `7`, `0x1A` → `26`, `4.5` → `4.5`) — so two writings of the same value normalize identically and comparison is numeric. A `date`'s normalized form is the source string **verbatim** (trimmed) — Myne does not reformat dates (no native timestamp parse).
- **No unicode normalization of values.** A `string` value's normalized form is the YAML-resolved value byte-for-byte — NFD combining sequences stay NFD (consistent with the §6 C1 byte-stability philosophy). The hex pins are the contract; JSON-string mirrors may be silently NFC-normalized by tooling.
- **Extraction + typing only.** No evaluatable expression, no computed/derived field, no formula (the anti-plugin / "not a database" commitment, `vision.md`, expressed at the type layer).

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/property-coercion.json`](../../schema/v0.1/property-coercion.json) (per-case shape). Every case carries a one-key YAML 1.2 mapping `prop: <value>` as `input_yaml_utf8_hex` (+ an `informational_yaml` mirror); the coercion target is the value of key `prop`. Two case kinds:

`case_kind: "scalar"`:

- `expected_type` — one of `string` / `number` / `date` / `checkbox` (a scalar never types as `list`).
- `expected_normalized_utf8_hex` — the canonical normalized form as exact UTF-8 bytes (string → resolved value verbatim, no NFC; number → canonical decimal of the parsed double; date → source string verbatim; checkbox → `true` / `false`).
- `informational_normalized` — optional human-readable mirror; cross-checked byte-equal against the hex.

`case_kind: "list"`:

- `expected_type` — const `list`.
- `expected_elements` — an array, one entry per sequence element, each independently coerced (`element_type` + `element_normalized_utf8_hex`). Empty array = the empty sequence `[]`.

## How to consume

A port runs each case:

1. Hex-decode `input_yaml_utf8_hex`, parse it as a YAML 1.2 **core-schema** document, take the value of key `prop`.
2. Run the port's property-coercion equivalent.
3. For `scalar`: assert the resulting type equals `expected_type` and the normalized **bytes** equal `expected_normalized_utf8_hex`. For `list`: assert type `list` and each element's `(type, normalized bytes)` in order.

Compare bytes, not parsed/normalized strings.

## Cases (31)

27 `scalar` + 5 `list`. The load-bearing tie-breaks:

| `vector_id` | pins |
|---|---|
| `number-float-trailing-zero` | `1.0` → number, normalized `1` (canonical decimal) |
| `string-numeric-quoted` | `"1.0"` → string (quoting demotes out of number) |
| `number-leading-zero` | `007` → number `7` (YAML core base-10 int; quote for a literal string) |
| `number-hex` | `0x1A` → number `26` (YAML core hex int) |
| `number-year-bare` | `2026` → number, **not** a date |
| `string-partial-date` | `2026-06` → string (not a complete full-date) |
| `checkbox-true-capitalized` | `True` → checkbox (core accepts `true`/`True`/`TRUE`) |
| `string-yes` / `string-on` | not YAML 1.2 core booleans → string |
| `string-bool-quoted` | `"true"` → string (quoting demotes out of checkbox) |
| `date-quoted-not-demoted` | `"2026-06-14"` → **date** (quoting does NOT demote a date) |
| `string-date-with-text` | `2026-06-14 deadline` → string (whole-string anchor) |
| `string-unicode-nfd` | NFD byte-transparency (no NFC normalization) |
| `list-mixed-types` | `[1, true, hello, 2026-06-14]` → `[number, checkbox, string, date]` |

`date-quoted-not-demoted` and the `number-year-bare`/`string-partial-date` pair are the most divergence-prone — they fail any implementation that conflates date recognition with the bool/number quoting rule, or that treats a bare integer as a year.

## Invariants the vectors pin

- **Single type per value.** Every value resolves to exactly one of the five types; the order of the Step-3 scalar rules (checkbox → number → date → string) is deterministic.
- **YAML 1.2 core semantics** for bool/number; **Myne RFC 3339 recognizer** for date on resolved strings.
- **Verbatim normalization** for number/date/string; canonical `true`/`false` for checkbox.
- **Per-element list coercion**, heterogeneous lists allowed, no list-of-lists in v0.1.

## What this does NOT cover

- **Querying / filtering / comparison operators** — the query-engine sub-feature's closed allow-list, not this layer.
- **Tag / folder pseudo-property identity** — reuses the existing §3.7.8 `foldSegment` rules, not the value-typing rules here.
- **Nested mappings** — coerce to `string` (best-effort), not a structured type in v0.1; not separately vectored beyond the documented rule.

## Regeneration

Hand-curated against §3.6.4 — **no Rust generator** (the coercion lives TS-side). The cases were assembled by a one-off script that hex-encodes a readable value→type table and self-checks every hex round-trip; the authored table is the source of truth. Conformance runs **TS-side**: the property-index Vitest runner hex-decodes each case, runs the live TS coercion, and asserts the type + normalized bytes. A failing vector in any port (or the desktop TS impl) is a stop-the-world bug — read why before regenerating.
