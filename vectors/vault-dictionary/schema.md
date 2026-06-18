# `vault-dictionary` vector schema

Pins the byte mapping of `protocol::encode_dictionary_blob_raw` + `protocol::decode_dictionary_blob` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)). The codec emits and consumes `vault.dictionary` — the v0.1 vault-level singleton encrypted **personal-dictionary** blob (`MYD1` magic; MYND-266) that lives next to `vault.meta` and carries the user's added-words set plus a vault-wide ignore set (the spell-check vocabulary) in encrypted form.

**Reference implementation:** `protocol/src/dictionary_blob.rs::{encode_dictionary_blob_raw, decode_dictionary_blob}`.
**Spec:** `docs/spec/protocol.md` § 9.13 (`vault.dictionary` outer envelope + opaque inner plaintext + AAD + size cap + fall-to-empty semantic) + § 3.4 registry entry `"myne.v0.1 personal-dictionary-key"` (the KEK role) + § 5.2 / § 9.13.3 `AAD_personal_dictionary_blob` formula (the AAD role `"personal-dictionary-blob"`).

## Opaque inner plaintext (unlike `vault-prefs`)

`vault.dictionary` is **payload-agnostic**: the inner plaintext is an opaque byte string the desktop layer produces (the `dict_format_version 0x01` format is a postcard-serialized `{personal_words, ignore_words}` struct — NFC-normalized, sorted, deduped word lists). The protocol crate owns only the crypto framing — magic, version bytes, AEAD, AAD — while the word-set serialization is the desktop layer's concern, versioned by the `dict_format_version` byte at offset 5. So, unlike `vault-prefs`, there is **no canonical-JSON inner shape** to pin: these vectors treat the plaintext as raw bytes and pin byte-transparency across the AEAD envelope. (Same posture as `vault-search` / `vault-workspace` / `vault-indices`; the five of them — prefs / search / workspace / indices / dictionary — are the vault-singleton blob family, one magic + one KEK + one AAD role each.)

## Two distinct load-bearing role strings

A porter must not conflate the two role strings this codec uses — they are different constants bound at different layers:

- **KEK (HKDF) role** `"personal-dictionary-key"` (23 ASCII bytes) — `personal_dictionary_key = DeriveSubkey(root_key, "personal-dictionary-key")` per § 3.2 + § 3.4. Pinned independently in the `hkdf-sha256` category.
- **AAD role** `"personal-dictionary-blob"` (24 ASCII bytes, length-prefixed) — the role inside `AAD_personal_dictionary_blob` per § 5.2 / § 9.13.3. Pinned independently in the `ctx-aead` category (`aad_canonical_bytes` case) and decomposed in `aad-construction`.

This `vault-dictionary` category exercises both end-to-end through the full encode/decode pipeline.

## Pipeline (as fixed by the reference implementation)

**Encrypt side** (`encode_dictionary_blob_raw`):

1. **Derive `personal_dictionary_key`** via the canonical helper: `derive_subkey(root_key, SubkeyRole::PersonalDictKey)`. HKDF-SHA-256 with `info = "myne.v0.1 personal-dictionary-key"`, `salt = zero_32`. Output is 32 bytes, deterministic.
2. **Build the AAD** via `aad_personal_dictionary_blob(account_number_hash)`: 53-byte concatenation of `"myne.v0.1|"` (10 B) + protocol_version `0x01` (1 B) + length-prefixed UTF-8 role `"personal-dictionary-blob"` (2-B BE length `0x0018` + 24 ASCII bytes = 26 B) + `account_number_hash` (16 B) per spec § 9.13.3.
3. **AEAD-seal** the opaque plaintext under `personal_dictionary_key` + `nonce` + AAD using XChaCha20-Poly1305 with the CTX (Chan & Rogaway 2022) outer commitment per spec § 5.1. Output is `plaintext.len() + 32` bytes (CTX tag is 32 bytes).
4. **Assemble the outer envelope** per spec § 9.13.1 fixed-offset layout (total `34 + ciphertext_len` bytes):
   - `[0..4]` magic `MYD1`
   - `[4]` protocol_version `0x01`
   - `[5]` dict_format_version `0x01` (inner word-set serialization format — a future client bumps this so an old decoder rejects cleanly → fall to the empty dictionary)
   - `[6..30]` `nonce` (24 B)
   - `[30..34]` `ciphertext_len` (big-endian uint32, `plaintext.len() + 32`)
   - `[34..]` ciphertext + 32-byte CTX commitment tag

**Decrypt side** (`decode_dictionary_blob`): the inverse — parse header, validate magic / protocol_version / **dict_format_version** (all three checked **before** any decryption per the § 9.13.1 magic-byte-before-decrypt discipline), read `nonce` + `ciphertext_len` (reject `> 4 MiB + 32` cap), rebuild AAD from `account_number_hash`, AEAD-decrypt. AEAD failure (wrong key / wrong AAD / tampered ciphertext) collapses to a single non-distinguishing `CannotDecrypt` per spec § 6.4.

The encode side is **pure and deterministic** given the caller-supplied `nonce` (no internal CSPRNG, no clock) — same inputs produce identical output bytes. Production callers use the `encode_dictionary_blob` convenience wrapper that pulls a nonce from `getrandom`; vectors pin a constant nonce for reproducibility.

## File layout

```
vectors/vault-dictionary/
├── schema.md              ← this file
└── v0.1/
    └── cases.json         ← array of 12 KAT cases (encrypt + decrypt + AEAD-fail + structural parse-fail)
```

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/vault-dictionary.json`](../../schema/v0.1/vault-dictionary.json) (per-case shape with a 4-way `case_kind` discriminator: `encrypt`, `decrypt_ok`, `decrypt_aead_fail`, `parse_fail`). Run the repository's documented `ajv-cli validate` command (see the root `README.md`) to verify any `cases.json` against the canonical schema. The sub-schema is authoritative — this document covers only the domain-specific semantics.

Four case kinds carry different field shapes:

- **`encrypt`** — pins the encoder's byte output. Inputs: `input_root_key_hex` (32 B; the runner derives `personal_dictionary_key` from it), `input_account_number_hash_hex` (16 B), `input_plaintext_hex` (opaque, may be empty), `input_nonce_hex` (24 B). Optional `informational_personal_dictionary_key_hex` carries the pre-computed KEK for reviewer convenience. Output: `expected_vault_dictionary_blob_hex` (the `MYD1` blob = 34-header + ciphertext + 32-CTX-tag).
- **`decrypt_ok`** — pins the decoder's successful round-trip. Inputs: `input_blob_hex` + correct `input_root_key_hex` + correct `input_account_number_hash_hex`. Output: `expected_plaintext_hex` (raw opaque bytes).
- **`decrypt_aead_fail`** — pins the **non-distinguishing** failure surface. Inputs: `input_blob_hex` + (intentionally wrong or tampered) inputs. Output: `expected_kind: "aead_fail"`. The decoder MUST collapse cross-vault (wrong `account_number_hash`), wrong-key, ciphertext-tamper, and CTX-tamper to a single indistinguishable failure (spec § 6.4). Fixtures carry the REAL key so a port that skips a check surfaces as a spurious `Ok` rather than a coincidental AEAD failure.
- **`parse_fail`** — pins a **distinguishable** pre-decryption structural reject. Inputs: `input_blob_hex` + `expected_error` (one of `magic_mismatch`, `protocol_version_mismatch`, `format_unsupported`, `header_too_short`, `ciphertext_truncated`, `ciphertext_len_overflow`). Unlike the AEAD-failure surface, these run **before** decryption and ARE distinguishable: a port must reject e.g. a future `dict_format_version` cleanly so the caller can fall back to the empty dictionary (§ 9.13.4). The Rust reference names the format-version reject `DictionaryBlobError::DictFormatUnsupported`; the vector string stays `format_unsupported` for cross-family consistency.

## How to consume

A port runs each case based on `case_kind`:

- **`encrypt`**: derive `personal_dictionary_key` from `input_root_key_hex`, invoke the port's `encode_dictionary_blob_raw` equivalent with the supplied nonce + plaintext + account hash, hex-encode, compare byte-equal to `expected_vault_dictionary_blob_hex`.
- **`decrypt_ok`**: derive the key, invoke the decoder, compare the recovered bytes to `expected_plaintext_hex`.
- **`decrypt_aead_fail`**: invoke the decoder, assert a non-distinguishing AEAD failure (the port's equivalent of `CannotDecrypt`). The port MUST NOT distinguish which arm fired.
- **`parse_fail`**: invoke the decoder, assert it returns the *specific* structural error named by `expected_error`, before any decryption. (Key/account-hash are irrelevant here — the reject precedes the AEAD stage.)

Any mismatch is a stop-the-world bug (per repo `README.md` "Cross-implementation discipline").

## Invariants the vectors pin

- **MYD1 magic + version + dict_format bytes.** Encoder output `[0..4]` = `MYD1`, `[4]` = `0x01`, `[5]` = `0x01`. Decoder rejects any other value at parse time before reaching AEAD verification — distinct, distinguishable errors (`parse_fail` cases). `MYD1` is its own blob family (one magic).
- **Header byte layout.** `[6..30]` = supplied `nonce` (24 B); `[30..34]` = `ciphertext_len` big-endian uint32; `[34..]` = ciphertext + 32-byte CTX tag.
- **Empty dictionary round-trips.** An empty plaintext (the fresh-vault / fall-to-empty default) encodes to a 66-byte blob (34 header + 0 plaintext + 32 CTX tag) and decodes back to empty.
- **AAD binds account identity, not items_key lineage.** Cross-vault attack (`decrypt-cross-vault-account-hash`): a blob encoded under vault A's `account_number_hash` MUST fail to decrypt under vault B's. The `personal_dictionary_key` derives from `root_key` independent of the `itemsKey` lineage.
- **Tamper rejection.** Flipping a single bit in the ciphertext / CTX tag MUST cause a non-distinguishing AEAD failure (CTX commitment construction).
- **Non-distinguishing AEAD surface vs distinguishable structural surface.** The two `decrypt_aead_fail` cases (tamper + cross-vault) MUST be indistinguishable from each other; the six `parse_fail` cases MUST each surface their specific structural reject. This split is the spec § 6.4 boundary-policy discipline applied to the vault-dictionary codec.
- **`dict_format_version` forward-migration.** The `parse-fail-format-unsupported` case pins that a v0.1 decoder rejects a higher `dict_format_version` cleanly (`DictFormatUnsupported`) so the caller falls back to the empty dictionary — the load-bearing forward hook (§ 9.13.4).
- **`ciphertext_len` overflow guard.** The `parse-fail-ciphertext-len-overflow` case pins that an absurd `ciphertext_len` (`0xFFFFFFFF`, far above the 4 MiB + 32 cap) is rejected before any slice indexing / allocation — a port MUST bound-check before allocating.

## Regeneration

If the reference implementation changes (must not happen byte-stably without a major-version event), regenerate this vector via:

```bash
cargo test -p protocol --lib emit_vault_dictionary_vectors_for_paste -- --ignored
cat /tmp/myne_vault_dictionary_vectors.json
# copy the full JSON verbatim into cases.json
```

The regenerator test lives in `protocol/src/dictionary_blob.rs::tests::emit_vault_dictionary_vectors_for_paste`. It is `#[ignore]`-gated so it never runs in default `cargo test` invocations.

## What this does NOT cover

- **Encode-only error modes.** `DictionaryBlobError::PlaintextTooLarge` (plaintext above the 4-MiB cap) and `DictionaryBlobError::RandomnessUnavailable` (OS CSPRNG failure in the non-`_raw` wrapper) are encode-side and cannot be expressed as a decode-a-blob case, so they are not vectored. The cap is pinned from the read direction by the `ciphertext-len-overflow` case instead.
- **The inner word-set serialization format.** That is the desktop layer's concern (client-opaque to this codec — the postcard `{personal_words, ignore_words}` struct); a future `dict_format_version` bump changes it without touching this crypto framing.
- **The Tauri-layer fall-back discipline** (absent file / truncated / AEAD-fail / unsupported-format all collapse to "the empty dictionary" at the plugin boundary) — wrapper behavior is verified by `src-tauri` integration tests, not cross-implementation vectors.
- **Schema evolution within v0.1.** A future `dict_format_version` lands as a new case rather than mutating existing entries (immutable `vector_id`s).
