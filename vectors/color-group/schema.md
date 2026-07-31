# `color-group` — color-groups array decoder

Pins `protocol::color_group::parse_color_groups_array` — the canonical-JSON
decoder for the `color_groups` array embedded inside `vault.prefs`.

- **Reference implementation:** `protocol/src/color_group.rs` (main `myne` repo)
- **Spec anchor:** `protocol.md` §9.5.2 (graph color groups; MYND-604)
- **Runner:** `protocol/tests/vectors.rs::color_group_vectors` (`cargo test -p protocol color_group_vectors`)
- **Generator:** `cargo test -p protocol --lib emit_color_group_vectors_for_paste -- --ignored` → `/tmp/myne_color_group_vectors.json`

## What this category pins

A color group is one graph-node coloring rule: `{id, name, color, enabled,
filter}`, where `filter` reuses the **same** `FilterNode` AST `saved_views`
persists (there is exactly one filter grammar in the crate). The `color_groups`
field of `vault.prefs` carries a JSON **array** of them. This category pins the
**decode** direction: input bytes → `(Vec<ColorGroup>, consumed)`.

The bytes are decoded only AFTER the surrounding `vault.prefs` AEAD has verified,
so a failure here is structural (the `prefs_key` holder wrote malformed JSON),
not adversarial. The category is therefore deterministic cross-client behavior in
the `saved-view` / `note-template` class — not a codec, not cryptographic in
itself — but it IS load-bearing for cross-implementation agreement.

## The defining divergence from `saved-view`

**Array position IS rule precedence (first match wins), so the array is
re-encoded in caller order VERBATIM — never sorted by `id`.** `saved_views` is
id-sorted on encode because its *display* order lives in the separate
`view_order` field; `color_groups` has no such field, so re-sorting would
silently corrupt precedence. A port that copies the `saved_views` id-sort here
is wrong, and the `multi-precedence-verbatim` case exists to catch exactly that.

## Case shape

```
cases[]:
  vector_id    "color-group/v0.1/<slug>"
  case_kind    "parse_ok" | "parse_fail"
```

### `parse_ok`

| field | meaning |
|---|---|
| `input_bytes_utf8_hex` | the input bytes (a JSON array, optionally with trailing bytes) |
| `expected_consumed_len` | bytes the decoder MUST report consumed — the array length, never including trailing bytes |
| `expected_canonical_utf8_hex` | the decoded set re-encoded via `canonical_encode_color_groups` |

The decoded rule-set is pinned **as its canonical re-encoding** rather than as a
parsed structure, so the vector stays language-agnostic. The canonical projection
captures every normalization a conformant decoder must converge to:

- **Verbatim precedence order**: the array is emitted in caller order, NOT
  id-sorted (`multi-precedence-verbatim`) — the inverse of `saved-view`.
- **NFC folding** of all strings on encode (the `nfc-fold-name` case).
- **Lexicographic per-object key order** (`color` < `enabled` < `filter` < `id`
  < `name`) + whitespace removal (`normalize-whitespace-and-key-order`).
- A **disabled** rule (`enabled: false`) is retained verbatim
  (`disabled-rule-retained`).

### `parse_fail`

| field | meaning |
|---|---|
| `input_bytes_utf8_hex` | the malformed/invalid input bytes |
| `expected_error` | one of `malformed` / `missing_field` / `wrong_type` / `unrepresentable` |

All five fields (`id` / `name` / `color` / `enabled` / `filter`) are required —
there are no optional sub-fields. `unrepresentable` is the closed-allow-list
reject inherited from the shared `FilterNode`: an unknown filter `kind` (a
formula/arithmetic node the anti-eval type cannot express) or an unknown
comparison operator.

## How to consume

For each `parse_ok` case: hex-decode `input_bytes_utf8_hex`, run your decoder,
assert it consumed exactly `expected_consumed_len` bytes, re-encode the decoded
set canonically, and assert it equals `expected_canonical_utf8_hex`
byte-for-byte. For each `parse_fail` case: assert your decoder rejects with the
named error.

## Invariants the vectors pin

- **Precedence is order, and order is preserved** — the load-bearing divergence
  from `saved-view` (`multi-precedence-verbatim`).
- The **cursor contract**: `consumed` reports only the array, so the embedding
  prefs codec can continue past it (`consumes-only-the-array-trailing`).
- The decoder tolerates insignificant whitespace but enforces the closed filter
  allow-list on decode.

## What this does NOT cover

The **encode** direction (`canonical_encode_color_groups` byte output) is pinned
transitively by the `parse_ok` re-encoding. The `vault.prefs` AEAD framing that
wraps this array is the `vault-prefs` category's job — where an `encrypt-populated`
case additionally pins `color_groups`' canonical **slot** inside the prefs blob.
The `color` token's resolution against the design-token palette is render-time
application behavior, out of scope here.
