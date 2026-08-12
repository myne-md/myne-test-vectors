# `vault-prefs` vector schema

Pins the byte mapping of `protocol::encode_prefs_blob_raw` + `protocol::decode_prefs_blob` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)). The codec emits and consumes `vault.prefs` — the v0.1 vault-level singleton encrypted-preferences blob (MYP1 magic) that lives next to `vault.meta` and carries user-controllable preferences (Phase 1: the screen-capture-protection toggle from MYND-55 / MYND-160).

**Reference implementation:** `protocol/src/prefs.rs::{encode_prefs_blob_raw, decode_prefs_blob}`.
**Spec:** `docs/spec/protocol.md` § 9.5 (`vault.prefs` outer envelope + inner canonical JSON + AAD + size cap + migration-less semantic) + § 3.4 registry entry `"myne.v0.1 prefs-key"` + § 5.2 `AAD_prefs_blob` formula.

## Pipeline (as fixed by the reference implementation)

**Encrypt side** (`encode_prefs_blob_raw`):

1. **Derive `prefs_key`** via the canonical helper: `derive_subkey(root_key, SubkeyRole::PrefsKey)`. HKDF-SHA-256 with `info = "myne.v0.1 prefs-key"`, `salt = zero_32`. Output is 32 bytes, deterministic. The same `root_key` always produces the same `prefs_key`.
2. **Canonical-JSON encode the `PrefsView`.** v0.1's schema (MYND-160 + MYND-183) carries `screen_capture_protection` (bool) plus the omitted-when-default optionals `deliberate_folders` (sorted-set array), `auto_lock_seconds` (integer), `editor_layout` (nested object), and the unified manual-sidebar sequences `favorites` + `item_order` (the latter **heterogeneous**: note UUIDs + canonical trailing-slash folder paths interleaved, replacing the pre-MYND-183 `note_order`/`folder_order` pair). Lexicographically sorted keys (`auto_lock_seconds < deliberate_folders < editor_layout < favorites < item_order < screen_capture_protection`), no whitespace, UTF-8 NFC strings, no floating-point per spec § 9.5.2; the **sequence** arrays (`favorites`, `item_order`) preserve element order verbatim while the `deliberate_folders` **set** is sorted + deduped. A screen-capture-only vault still renders as `{"screen_capture_protection":true}` (all optionals omitted), byte-identical to a pre-MYND-183 blob.
3. **Build the AAD** via `aad_prefs_blob(account_number_hash)`: 39-byte concatenation of `"myne.v0.1|"` (10 B) + protocol_version `0x01` (1 B) + length-prefixed UTF-8 role `"prefs-blob"` (2-B BE length + 10 ASCII bytes = 12 B) + `account_number_hash` (16 B) per spec § 9.5.3.
4. **AEAD-seal** the canonical JSON plaintext under `prefs_key` + `prefs_nonce` + AAD using XChaCha20-Poly1305 with the CTX (Chan & Rogaway 2022) outer commitment per spec § 5.1. Output is `plaintext.len() + 32` bytes (CTX tag is 32 bytes).
5. **Assemble the outer envelope** per spec § 9.5.1 fixed-offset layout (total `34 + ciphertext_len` bytes):
   - `[0..4]` magic `MYP1`
   - `[4]` protocol_version `0x01`
   - `[5]` reserved `0x00`
   - `[6..30]` `prefs_nonce` (24 B)
   - `[30..34]` `ciphertext_len` (big-endian uint32, `plaintext.len() + 32`)
   - `[34..]` ciphertext + 32-byte CTX commitment tag

**Decrypt side** (`decode_prefs_blob`): the inverse — parse header, validate magic / version / reserved, read `prefs_nonce` + `ciphertext_len` (reject `> 65536 + 32` cap per spec § 9.5.4), rebuild AAD from `account_number_hash`, AEAD-decrypt, canonical-JSON parse. AEAD failure (wrong key / wrong AAD / tampered ciphertext) collapses to a single non-distinguishing `CannotDecrypt` per spec § 6.4 + § 9.5.5.

The encode side is **pure and deterministic** given the caller-supplied `prefs_nonce` (no internal CSPRNG, no clock) — same inputs produce identical output bytes. Production callers use the `encode_prefs_blob` convenience wrapper that pulls a nonce from `getrandom`; vectors pin a constant nonce for reproducibility.

## File layout

```
vectors/vault-prefs/
├── schema.md              ← this file
└── v0.1/
    └── cases.json         ← array of 52 KAT cases (encrypt + decrypt + failure-mode families, grown additively per feature — MYND-183/110/191/184/185/205/58 + the MYND-227 parse_fail family)
```

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/vault-prefs.json`](../../schema/v0.1/vault-prefs.json) (per-case shape with a 4-way `case_kind` discriminator: `encrypt`, `decrypt_ok`, `decrypt_aead_fail`, `parse_fail`). Run the repository's documented `ajv-cli validate` command (see the root `README.md`) to verify any `cases.json` against the canonical schema. The sub-schema is authoritative — this document covers only the domain-specific semantics.

Four case kinds carry different field shapes:

- **`encrypt`** — pins the encoder's byte output. Inputs: `input_root_key_hex` (32 B), `input_account_number_hash_hex` (16 B), `input_plaintext_canonical_utf8_hex`, `input_prefs_nonce_hex` (24 B). Optional `informational_prefs_key_hex` carries the pre-computed HKDF output for reviewer convenience. Output: `expected_prefs_blob_hex` (the `MYP1` blob = 34-byte header + ciphertext + 32-byte CTX tag; e.g. 100 bytes for the single-field default, larger for the fully-populated case).
- **`decrypt_ok`** — pins the decoder's successful round-trip. Inputs: `input_blob_hex` + correct `input_root_key_hex` + correct `input_account_number_hash_hex`. Outputs: `expected_plaintext_canonical_utf8_hex` (the canonical re-encode of the decoded view) and `expected_prefs_view` (the parsed `PrefsView` object — implementations parse the recovered JSON and assert structural equality). For ordinary round-trips the canonical re-encode equals the sealed plaintext byte-for-byte; for the forward-compat tolerance case (`decrypt-unknown-field-tolerated`, spec § 9.5.2) the sealed plaintext carries an unknown additive field (carried verbatim in the optional `informational_sealed_plaintext_utf8_hex`) that decoders MUST ignore — there the comparison is against the re-encode WITHOUT the unknown field.
- **`decrypt_aead_fail`** — pins the non-distinguishing failure surface. Inputs: `input_blob_hex` + (intentionally wrong or tampered) `input_root_key_hex` + `input_account_number_hash_hex`. Output: `expected_kind: "aead_fail"`. The decoder MUST collapse cross-vault, wrong-AAD, ciphertext-tamper, and CTX-tamper to a single indistinguishable failure (spec § 6.4 + § 9.5.5).
- **`parse_fail`** (MYND-227) — pins the **distinguishable** reject surface, one case per `PrefsBlobError` variant reachable from a decode: the six pre-AEAD structural rejects (`header_too_short`, `magic_mismatch`, `protocol_version_mismatch`, `reserved_byte_nonzero`, `ciphertext_len_overflow`, `ciphertext_truncated`) plus the post-AEAD `plaintext_parse_failed` (a validly-sealed blob whose inner plaintext is not canonical JSON). `expected_error` names the variant. The cases carry the REAL `root_key` deliberately: header bytes are not AAD-bound, so a port that skips a structural check decrypts the tampered blob successfully and returns Ok — which the runner catches; a dummy key would let the same buggy port pass silently. Callers MAY collapse all of these to `PrefsView::default()` per § 9.5.5 — the distinguishability is an observability contract, not a UX one.

## How to consume

A port runs each case based on `case_kind`:

- **`encrypt`**: decode every `input_*_hex`, build a `PrefsView` from the implied plaintext shape, invoke the port's `encode_prefs_blob_raw` equivalent with the supplied nonce, hex-encode the output, compare byte-equal to `expected_prefs_blob_hex`. Any non-equal result is a stop-the-world bug (per repo `README.md` "Cross-implementation discipline").
- **`decrypt_ok`**: decode `input_blob_hex` + key + account hash, invoke the port's `decode_prefs_blob` equivalent, parse the recovered JSON tolerantly (unknown additive fields ignored per § 9.5.2), compare the parsed view to `expected_prefs_view`, AND canonical-re-encode the parsed view and compare byte-equal to `expected_plaintext_canonical_utf8_hex`. (Comparing the raw recovered plaintext directly works for ordinary round-trips but NOT for the tolerance case — compare the re-encode.)
- **`decrypt_aead_fail`**: decode inputs, invoke decoder, assert it returns a non-distinguishing AEAD failure (whatever the port's equivalent is — `Err(AeadFailure)`, exception, error code). The port MUST NOT distinguish which failure arm fired beyond "this blob cannot be decrypted with these inputs".
- **`parse_fail`**: decode inputs, invoke decoder, assert the port surfaces its equivalent of the named `expected_error` variant — and in particular does NOT return Ok (a port that skips the named check will decode the tampered blob successfully) and does NOT report its AEAD-failure error for `plaintext_parse_failed` (two distinct failure surfaces).

## Why hex, not JSON-native UTF-8 strings or numeric arrays

JSON natively supports UTF-8 strings via `\uXXXX` escapes, but escape behavior varies subtly across language libraries. JSON also supports numeric arrays for byte sequences, but byte-array parsing varies and is awkward to diff. Hex-encoding the raw bytes eliminates parser-side transformations: bytes that go in are exactly the bytes the reference implementation saw.

## Invariants the vectors pin

For the 52 cases in `cases.json`:

- **Every field is pinned beside BOTH of its canonical neighbours, not alone.** `vault.prefs` is canonical JSON, so **key order is part of the bytes**: a port that emits a correct value in the wrong slot produces a blob the reference decoder rejects, and a case that pinned the value alone would not notice. The `encrypt-datetime-quartet` pair is the worked example — `auto_tag_enabled`, `date_format`, `time_format` and `timezone` each sit between their two neighbours (`auto_lock_seconds` < `auto_tag_enabled` < `bar_items_order`; `daily_template` < `date_format` < `deliberate_folders`; `theme` < `time_format` < `timezone` < `transcription_enabled`). Those four shipped with no cross-implementation vector at all until that pair landed, exercised only by `#[cfg(test)]` unit tests that no port can run.

- **MYP1 magic + version + reserved bytes.** Encoder output `[0..4]` = `MYP1`, `[4]` = `0x01`, `[5]` = `0x00`. Decoder rejects any other value at parse time before reaching AEAD verification.
- **Header byte layout.** Encoder output `[6..30]` = supplied `prefs_nonce` (24 B); `[30..34]` = `ciphertext_len` as big-endian uint32; `[34..]` = ciphertext + 32-byte CTX tag.
- **Canonical JSON discipline.** The baseline v0.1 plaintext is exactly `{"screen_capture_protection":true}` (34 bytes) or `{"screen_capture_protection":false}` (35 bytes); later cases pin the additive v0.1.x fields in lexicographic key order with omit-when-default semantics (see spec § 9.5.2 for the full key registry). No whitespace, no key reordering, no trailing characters, no leading BOM.
- **MYND-205 `label` field.** The `encrypt-labeled` / `decrypt-labeled` pair pins the per-vault display name: NFC normalization on encode (the generator supplies the label in NFD; the canonical bytes are NFC) and the canonical position `item_order` (i) < `label` (l) < `screen_capture_protection` (s).
- **Forward-compat tolerance (§ 9.5.2).** `decrypt-unknown-field-tolerated` pins that an unknown additive field in the sealed plaintext is ignored, not rejected — the load-bearing schema-evolution hook for v0.1.x additive fields.
- **MYND-58 `last_backup_at` field.** The `encrypt-last-backup-at` / `decrypt-last-backup-at` pair pins the epoch-seconds backup timestamp (vault-backup-restore epic): the relative order of the two 'l' keys (`label` [lab] < `last_backup_at` [las] < `screen_capture_protection`) and the bare-decimal u64 encoding (no trailing `.0`; floats FORBIDDEN).
- **AAD binds account identity, not items_key lineage.** Cross-vault attack (case 4): a blob encoded under vault A's `account_number_hash` MUST fail to decrypt under vault B's `account_number_hash`, even if `prefs_key` somehow matched. The AAD difference is structural defense alongside the key mismatch.
- **Tamper rejection (case 5).** Flipping a single bit anywhere in the ciphertext / CTX tag MUST cause the decoder to return a non-distinguishing AEAD failure — the CTX commitment construction guarantees this without depending on the underlying Poly1305 tag's malleability properties.
- **Non-distinguishing failure surface (cases 4 + 5).** A port's decoder MUST emit identical observable behavior for cross-vault AND tampered cases — both are "AEAD failure"; implementations MUST NOT leak which failure mode triggered.
- **Distinguishable structural rejects (MYND-227, `parse-fail-*`).** Each of the seven `PrefsBlobError` decode rejects is pinned individually — magic / version / reserved / length / cap checks all run before the AEAD stage (the `ciphertext_len_overflow` case pins the § 9.5.4 64 KiB + 32 cap from the read direction with a 34-byte fixture: the header advertises `0xffffffff` without carrying it, so a port MUST bound-check before allocating), and `plaintext_parse_failed` is distinct from `CannotDecrypt` (the tag check already proved key possession).

## Regeneration

If the reference implementation changes (must not happen byte-stably without a major-version event), regenerate this vector via:

```bash
cargo test -p protocol --lib emit_prefs_vectors_for_paste -- --ignored
cat /tmp/myne_vault_prefs_vectors.txt
# paste hex outputs into cases.json

# MYND-205 label + §9.5.2 tolerance cases (separate per-feature generator):
cargo test -p protocol --lib emit_label_and_tolerance_prefs_vectors -- --ignored
cat /tmp/myne_vault_prefs_label_vectors.txt

# MYND-58 last_backup_at cases (separate per-feature generator):
cargo test -p protocol --lib emit_last_backup_at_prefs_vectors -- --ignored
cat /tmp/myne_vault_prefs_last_backup_vectors.txt

# MYND-227 structural parse_fail cases (separate per-feature generator, JSON array fragment):
cargo test -p protocol --lib emit_prefs_parse_fail_vectors -- --ignored
cat /tmp/myne_vault_prefs_parse_fail_vectors.json
```

The regenerator tests live in `protocol/src/prefs.rs` (`emit_prefs_vectors_for_paste` for the original case set; `emit_decorations_prefs_vector`, `emit_label_and_tolerance_prefs_vectors`, `emit_last_backup_at_prefs_vectors`, and `emit_prefs_parse_fail_vectors` for the later per-feature additions — the original generator stays stable as features land). All are `#[ignore]`-gated so they never run in default `cargo test` invocations. These are append generators: **splice** the new blocks/objects into `cases.json` by hand; never regenerate-and-replace the whole file.

## What this does NOT cover

- The full `vault.meta` serialization vectors (`vectors/vault_meta_rewrite/`) — those pin the password-rewrite path; this directory pins only the standalone vault.prefs codec.
- The Tauri-layer `prefs_read` / `set_screen_capture_protection` IPC commands — those wrap this codec with Slot::Unlocked-gating, account-hash extraction from vault.meta, and atomic disk write. Wrapper-layer behavior is verified by `src-tauri/src/plugins/prefs/commands.rs` integration tests, NOT cross-implementation vectors (the wrapper is Rust/Tauri-specific).
- The native `screen_capture::apply` side-effect — platform-specific, no cross-implementation contract.
- Exhaustive schema-evolution permutations. The `decrypt-unknown-field-tolerated` case pins the § 9.5.2 unknown-field tolerance principle with one representative future field; each shipped v0.1.x field addition extends this vector set with new cases rather than mutating existing entries (immutable `vector_id`s).
- **Encode-side rejects.** `CiphertextLenOverflow` on the encode path (oversized plaintext) and `RandomnessUnavailable` cannot be expressed as decode-a-blob cases and would need multi-MB fixtures; the 64 KiB cap is pinned from the read direction by the `parse-fail-ciphertext-len-overflow` case instead (deliberate, MYND-227).
