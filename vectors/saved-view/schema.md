# `saved-view` — saved-views array decoder

Pins `protocol::saved_view::parse_saved_views_array` — the canonical-JSON decoder
for the `saved_views` array embedded inside `vault.prefs`.

- **Reference implementation:** `protocol/src/saved_view.rs` (main `myne` repo)
- **Spec anchor:** `protocol.md` §9.5.2 (Saved Views; MYND-271)
- **Runner:** `protocol/tests/vectors.rs::saved_view_vectors` (`cargo test -p protocol saved_view_vectors`)
- **Generator:** `cargo test -p protocol --lib emit_saved_view_vectors_for_paste -- --ignored` → `/tmp/myne_saved_view_vectors.json` for the whole-file cases; `cargo test -p protocol --lib emit_saved_view_opaque_vectors_for_paste -- --ignored` → `/tmp/myne_saved_view_opaque_vectors.json` for the `view-manifests` and opaque-entry cases, a JSON fragment SPLICED into the existing `cases` array rather than replacing the file

## What this category pins

A Saved View is the persisted definition of one query/render view. The
`saved_views` field of `vault.prefs` carries a JSON **array** of them. This
category pins the **decode** direction: input bytes → `(Vec<SavedView>,
retained entries, consumed)`. The third element is the forward-compat half: an
entry a build cannot represent is kept verbatim rather than rejected.

The bytes are decoded only AFTER the surrounding `vault.prefs` AEAD has verified,
so a failure here is structural (the `prefs_key` holder wrote malformed JSON),
not adversarial. The category is therefore deterministic cross-client behavior
in the `note-template` / `property-coercion` class — not a codec, not
cryptographic in itself — but it IS load-bearing for cross-implementation
agreement: every independent client must decode the same bytes to the same
view-set and the same consumed length.

## Case shape

```
cases[]:
  vector_id    "saved-view/v0.1/<slug>"
  case_kind    "parse_ok" | "parse_ok_opaque" | "parse_fail"
```

### `parse_ok`

| field | meaning |
|---|---|
| `input_bytes_utf8_hex` | the input bytes (a JSON array, optionally with trailing bytes) |
| `expected_consumed_len` | bytes the decoder MUST report consumed — the array length, never including trailing bytes |
| `expected_canonical_utf8_hex` | the decoded set re-encoded via `canonical_encode_saved_views` |

The decoded view-set is pinned **as its canonical re-encoding** rather than as a
parsed structure, so the vector stays language-agnostic (no struct shape on the
wire). The canonical projection captures every normalization a conformant
decoder must converge to:

- **NFC folding** of all strings on decode (the `nfc-fold-name` case).
- **Lexicographic key order** + whitespace removal (the
  `normalize-whitespace-and-key-order` case — input and projection differ).
- **id-sort**: the array is emitted sorted by `id` regardless of caller order
  (`multi-sorted-by-id`).
- **Unknown-field drop**: a future key is ignored (forward-compat) and absent
  from the projection (`unknown-field-tolerated`).

### `parse_ok_opaque`

A decode that SUCCEEDS while retaining one or more entries this build cannot
represent: `view_type` outside `{table,cards,list}` (a newer client's), an
operator outside the closed set, or an entry whose `id` is not a string.

Retention is what keeps one newer client's view from taking the whole
`vault.prefs` blob down the §9.5.5 fall-back path, and from being pushed back to
every device as defaults on the next write. Three properties are pinned, one
case each: the retained entry's raw bytes come back unchanged (it is NOT
re-serialised, so a non-canonical key order in the input survives), the id sort
spans retained and represented entries together (a retained entry can sort ahead
of a represented one, which is why the sort cannot be per-kind), and an
unrepresentable entry whose `id` is not a string is still retained, sorting
first with an empty sort key rather than being dropped.

| field | meaning |
|---|---|
| `input_bytes_utf8_hex` | the input bytes (a JSON array carrying at least one unrepresentable entry) |
| `expected_consumed_len` | bytes the decoder MUST report consumed — still the array length |
| `expected_canonical_utf8_hex` | the decoded set re-encoded canonically with the retained entries spliced back in verbatim |
| `expected_opaque_ids` | the `id` of every retained entry, in emit order; a non-string `id` is pinned as `""` |

### `parse_fail`

| field | meaning |
|---|---|
| `input_bytes_utf8_hex` | the malformed/invalid input bytes |
| `expected_error` | one of `malformed` / `missing_field` / `wrong_type` / `unrepresentable` |

`unrepresentable` is the closed-allow-list reject: a `map` view type (no Maps
plugin in v1), an unknown filter `kind` (a formula/arithmetic node the anti-eval
type cannot express), or an unknown comparison operator.

## How to consume

For each `parse_ok` case: hex-decode `input_bytes_utf8_hex`, run your decoder,
assert it consumed exactly `expected_consumed_len` bytes, re-encode the decoded
set canonically, and assert it equals `expected_canonical_utf8_hex` byte-for-byte.
For each `parse_ok_opaque` case: the same three assertions, plus that the ids of
the entries you retained equal `expected_opaque_ids` in order. That last one
names WHICH entries were retained and in what order, so a port that drops one and
keeps another fails with a readable diff rather than only a byte mismatch.
For each `parse_fail` case: assert your decoder rejects with the named error.

## Invariants the vectors pin

- The **cursor contract**: `consumed` reports only the array, so the embedding
  prefs codec can continue past it (`consumes-only-the-array-trailing`).
- The decoder tolerates insignificant whitespace and unknown keys but bans JSON
  floats (`exponent-float-banned`).
- The closed allow-list is enforced on decode, not deferred to a later layer.

## What this does NOT cover

The **encode** direction (`canonical_encode_saved_views` byte output) is pinned
transitively by the `parse_ok` re-encoding, not as a standalone encode case. The
`vault.prefs` AEAD framing that wraps this array is the `vault-prefs` category's
job. The query-engine semantics of a filter AST (what notes a `Property` leaf
selects) are application behavior, out of scope here.
