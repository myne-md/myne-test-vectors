# `aad-construction` vector schema

Pins the **per-call-site AAD framing decomposition** for every `pub aad_*` builder in `protocol/src/aead.rs` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)).

**Reference implementation:** `protocol/src/aead.rs::{aad_password_wrap, aad_recovery_wrap, aad_vault_items_key_wrap, aad_note_item_key_wrap, aad_prefs_blob, aad_search_index_blob, aad_workspace_blob, aad_note_content_encrypt}`.
**Spec:** `docs/spec/protocol.md` § 5.2 (AAD construction).

## Relationship to `ctx-aead` (read this first)

The 8 `aad_*` builders are **already pinned** byte-for-byte in the [`ctx-aead`](../ctx-aead/) category via its `aad_canonical_bytes` cases. This category does **not** duplicate that — it adds one layer on top:

> `ctx-aead` pins *what bytes* each builder produces. `aad-construction` pins *how those bytes are framed* — the ordered segment decomposition: `aad_prefix` → `protocol_version` → (`role_length_be16` + `role_label`) → bound parameters, each with its byte offset and length.

The marginal coverage: a port that produces the right total bytes but assembles them from the wrong structural pieces — forgets the `u16` big-endian length-prefix on a role label, swaps field order, or mislabels which bound value goes where — can still match the byte total by coincidence on a single fixture, but its segment offsets will not line up. The `expected_aad_hex` field here is identical to the bytes `ctx-aead` pins (intentional cross-check); the `informational_aad_segments` array is the new contract.

## The framing (spec § 5.2)

Every Myne v0.1 AAD begins with the 10-byte prefix `"myne.v0.1|"` and the 1-byte `protocol_version` (`0x01`). After that, builders diverge:

| builder | bytes | segments |
|---|---|---|
| `aad_password_wrap` | 37 | prefix, version, role_length_be16, role_label(`password-wrap`), kdf_params_canonical(11) |
| `aad_recovery_wrap` | 37 | prefix, version, role_length_be16, role_label(`recovery-wrap`), kdf_params_canonical(11) |
| `aad_vault_items_key_wrap` | 27 | prefix, version, items_key_id(16) |
| `aad_note_item_key_wrap` | 43 | prefix, version, items_key_id(16), note_uuid(16) |
| `aad_prefs_blob` | 39 | prefix, version, role_length_be16, role_label(`prefs-blob`), account_number_hash(16) |
| `aad_search_index_blob` | 46 | prefix, version, role_length_be16, role_label(`search-index-blob`), account_number_hash(16) |
| `aad_workspace_blob` | 43 | prefix, version, role_length_be16, role_label(`workspace-blob`), account_number_hash(16) |
| `aad_note_content_encrypt` | 58 | prefix, version, items_key_id(16), note_uuid(16), content_type_length_be16, content_type(`text/markdown`) |

Note the two framing styles: the **wrap/blob** AADs carry a `u16`-be **length-prefixed role label** (so the role is unambiguous and length-bound); the **items-key / note** AADs bind raw fixed-width IDs with **no** role label (the field positions are themselves the structure). `aad_note_content_encrypt` mixes both — fixed-width IDs plus a length-prefixed `content_type`.

## JSON shape

The contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level) and [`schema/v0.1/aad-construction.json`](../../schema/v0.1/aad-construction.json). Single case shape; `input_builder_name` is the discriminator. Each case:

- `input_builder_name` — which builder (enum of 8).
- builder-specific inputs — `input_kdf_params` (wrap builders), `input_items_key_id_hex` / `input_note_uuid_hex` (note/items builders), `input_account_number_hash_hex` (blob builders).
- `expected_aad_hex` — the full AAD bytes (== the `ctx-aead` pin for the same builder).
- `informational_aad_segments` — ordered `{ offset, length, label, bytes_hex }` decomposition. The bytes concatenate to `expected_aad_hex`; offsets are contiguous from 0; each `length == bytes_hex.length / 2`.

## How to consume

For each case, a port:

1. Dispatches on `input_builder_name`, decodes the matching `input_*` fields, and invokes its equivalent builder.
2. Compares the output byte-equal to `expected_aad_hex`.
3. Validates the segments: walk `informational_aad_segments` in order asserting `offset` is contiguous (starts at 0, each next `offset == prev offset + prev length`), `length == bytes_hex` byte count, and the concatenation of all `bytes_hex` equals `expected_aad_hex`. A port MAY additionally assert its own assembly emits the same labeled segments in the same order — that is the structural check this category exists for.

## Invariants the vectors pin

- **Universal prefix.** Every AAD starts with `aad_prefix` = `6d796e652e76302e317c` (`"myne.v0.1|"`) then `protocol_version` = `01`. Drift in either breaks cross-major separation (§ 10) and is caught on every case.
- **Length-prefixed roles are `u16` big-endian.** The `role_length_be16` segment (e.g. `000d` for the 13-byte `password-wrap`, `0011` for the 17-byte `search-index-blob`) precedes every role/`content_type` label. A port that drops the prefix or uses little-endian fails the segment check.
- **Role separation.** `password-wrap` vs `recovery-wrap`, and `prefs-blob` vs `search-index-blob` vs `workspace-blob`, differ ONLY in the `role_label` segment — the decomposition makes that the single distinguishing field explicit.
- **Field order and width.** `items_key_id` (16) always precedes `note_uuid` (16) where both appear; `content_type` is the trailing length-prefixed segment in `aad_note_content_encrypt`.
- **Coverage.** The harness asserts all 8 `pub aad_*` builders have a case — adding a 9th builder without a vector fails the suite.

## The v0.2 side (`v0.2/cases.json`)

This category has a second version directory, and it is not a copy of the first.

`v0.2/` pins the four **next-major** vault-scoped builders —
`aad_password_wrap_v0_2`, `aad_recovery_wrap_v0_2`,
`aad_vault_items_key_wrap_v0_2` and the generic `aad_singleton_blob_v0_2`. Every
one of them opens with the 10-byte prefix `"myne.v0.2|"` and the `0x02`
`protocol_version` byte, both listed as **major-version-event constants** in the
main repo's root `CLAUDE.md`. Those 11 bytes are the whole of the §10.5
anti-downgrade separation: the same role under the two majors yields different
AAD, which is why role strings are safely reused across versions.

Two structural differences are easy to mis-port:

1. **There are no per-family singleton builders in v0.2.** No
   `aad_prefs_blob_v0_2`. The §10.4.1 whole-vault migration walks the blob
   families and rebuilds each one's AAD from its role string through the single
   generic `aad_singleton_blob_v0_2`, so **the role is an input** rather than
   baked into a function name. The v0.2 cases therefore carry
   `input_role_label`, and at least two roles of *different lengths* are pinned
   — same-length roles cannot catch an implementation that hardcoded the `u16`
   length prefix instead of computing it.
2. **The two note-scoped v0.2 builders are not here.** `aad_nc05_item_key_wrap`
   and `aad_nc05_content_encrypt` are pinned by
   [`crdt-note-blob`](../crdt-note-blob/schema.md)'s own `aad_construction`
   cases. One home per builder — duplicating them would create a second place to
   forget.

On `input_kdf_params` in the two wrap cases: v0.2's own KDF is **not allocated
yet**, so those cases carry the v0.1 canonical block. What they pin is the
*framing* — that the param block is bound at all, and at which offset — not a
v0.2 parameter choice. When v0.2 picks its KDF the values change and the offsets
do not.

Schema: [`schema/v0.2.json`](../../schema/v0.2.json) +
[`schema/v0.2/aad-construction.json`](../../schema/v0.2/aad-construction.json).
Generator: `emit_aad_construction_v0_2_vectors_for_paste` (`protocol/src/aead.rs`).
Runner: `aad_construction_v0_2_vectors` (`protocol/tests/vectors.rs`).

## What this does NOT cover

- **The CTX/AEAD seal itself.** How the AAD is hashed into the Poly1305 tag and the CTX commitment is pinned by `ctx-aead`'s `encrypt` / `decrypt_ok` / `decrypt_aead_fail` cases, not here.
- **`kdf_params` byte serialization internals.** The 11-byte canonical block is pinned as one `kdf_params_canonical` segment; its field-level byte layout is pinned by `argon2id` + `vault-meta`.
- **Non-canonical inputs.** One representative fixture per builder; exhaustive input fuzzing is out of scope (the framing is input-length-invariant for fixed-width fields).

## Regeneration

```bash
cargo test -p protocol --lib emit_aad_construction_vectors_for_paste -- --ignored
cp /tmp/myne_aad_construction_vectors.json vectors/aad-construction/v0.1/cases.json
```

The generator lives in `protocol/src/aead.rs::tests::emit_aad_construction_vectors_for_paste`. It slices each builder's real output into the labeled segments and asserts the segment lengths sum to the AAD length before emitting — so a regeneration where the decomposition and the builder disagree fails loudly. Decision-level conformance runs via `cargo test -p protocol --test vectors aad_construction_vectors`.
