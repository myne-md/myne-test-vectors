# `frontmatter-surgery` — frontmatter field patchers

Pins the pure byte-level frontmatter field patchers in `src/dialect/surgery.ts`.

- **Reference implementation:** `src/dialect/surgery.ts` (main `myne` repo)
- **Spec anchor:** `markdown-dialect.md` (frontmatter field surgery)
- **Runner:** `tests/dialect/frontmatterSurgery.test.ts` (`pnpm test`) — TS-side, no Rust generator
- **Generator:** `GEN_FM_SURGERY=1 pnpm vitest run tests/dialect/_genFrontmatterSurgeryVectors.test.ts`

## What this category pins

Each patcher is a pure `Uint8Array → Uint8Array` transform that inserts,
replaces, or removes **one** frontmatter field while leaving the rest of the note
(body, other fields, YAML formatting) byte-stable. They are the lowest-level
note-mutation primitive — every "set the folder", "mark trashed", "stamp
updated" operation routes through one. A port that emits different bytes here
silently diverges on disk.

This is deterministic cross-client behavior — not a codec, not cryptographic —
but load-bearing for byte-level agreement.

## Case shape

```
cases[]:
  vector_id                 "frontmatter-surgery/v0.1/<slug>"
  case_kind                 one of the eight patchers (below)
  input_plaintext_utf8_hex  the note plaintext, exact UTF-8 bytes (hex)
  expected_output_utf8_hex  the patched bytes (hex) — a fixed point (idempotent)
  informational_plaintext / informational_output   human-readable mirrors
```

| `case_kind` | patcher | extra input field |
|---|---|---|
| `updated` | `patchUpdatedField(bytes, iso)` | `input_iso` (string) |
| `title` | `patchTitleField(bytes, value)` | `input_value_utf8_hex` |
| `folder` | `patchFolderField(bytes, value\|null)` | `input_value_utf8_hex`? |
| `trashed` | `patchTrashedField(bytes, value\|null)` | `input_value_utf8_hex`? |
| `archived` | `patchArchivedField(bytes, value\|null)` | `input_value_utf8_hex`? |
| `template` | `patchTemplateField(bytes, value\|null)` | `input_value_utf8_hex`? |
| `folder-index` | `patchFolderIndexField(bytes, on)` | `input_flag` (bool) |
| `generic` | `patchFrontmatterField(bytes, key, value\|null)` | `input_key` + `input_value_utf8_hex`? |

**Null encoding.** For the nullable patchers (`folder` / `trashed` / `archived`
/ `template` / `generic`), an **absent** `input_value_utf8_hex` encodes `null`
(remove the field); a present value encodes the set-value.

## How to consume

Hex-decode `input_plaintext_utf8_hex`, apply the named patcher with the case's
argument(s), and assert the output equals `expected_output_utf8_hex` byte-for-byte.
The runner additionally re-checks the two at-mint invariants against your code:

- **Purity** — the input buffer is untouched after the call.
- **Idempotency** — applying the same patch to the output reproduces it exactly.

## What this does NOT cover

`patchFirstH1Line`, `patchSourceRanges`, and `buildInitialPlaintext` are out of
scope for this round (this category is the field-level patchers). The YAML
*reading* helpers (`readTitleFromFmBody`, …) are the inverse direction, also out
of scope here.
