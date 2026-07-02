# `embeddings` vector schema

Pins the byte mapping of `protocol::encode_embeddings_blob_raw` + `protocol::decode_embeddings_blob` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)). The codec emits and consumes `vault.embeddings` — the v0.1 vault-level singleton encrypted **on-device AI-embedding-index** blob (`MYR1` magic; MYND-453) that lives next to `vault.meta` and carries the semantic-search / Q&A dense vectors, a `{note_id → content_nonce}` staleness manifest, and a trailing model-identity tuple `{model_hash, embedding_dim, chunker_version}` in encrypted form.

**Reference implementation:** `protocol/src/embeddings_blob.rs::{encode_embeddings_blob_raw, decode_embeddings_blob}`.
**Spec:** `docs/spec/protocol.md` § 9.17 (`vault.embeddings` outer envelope + engine-opaque inner plaintext + AAD + size cap + fall-to-rebuild semantic) + § 3.4 registry entry `"myne.v0.1 ai-embeddings-key"` (the KEK role) + § 5.2 / § 9.17.3 `AAD_ai_embeddings_blob` formula (the AAD role `"ai-embeddings-blob"`).

## Distinct blob family from `vault.search` and `vault.indices`

`MYR1` / `vault.embeddings` (dense AI vectors), `MYS1` / `vault.search` (BM25 full-text) and `MYI1` / `vault.indices` (structured metadata) are **separate blob families** — one magic, one KEK role (`ai-embeddings-key` vs `search-index-key` vs `vault-indices-key`), and one AAD role (`ai-embeddings-blob` vs `search-index-blob` vs `vault-indices-blob`) each. A blob of one family can never AEAD-verify under another's key or AAD. (This category includes a unit-level pin — `indices_key_cannot_open_embeddings_blob` — in the reference crate; the cross-family separation is load-bearing because one-magic/one-KEK-per-family is the §10.3 discipline.)

## Opaque inner plaintext (unlike `vault-prefs`)

`vault.embeddings` is **payload-agnostic**: the inner plaintext is an opaque byte string the desktop layer produces (the serialized embedding store — dense vectors + `{note_id → content_nonce}` manifest + model-identity tuple, Build-B: TS/engine types, Rust stores opaque). The protocol crate owns only the crypto framing — magic, version bytes, AEAD, AAD — while the vector-store serialization (the inner **postcard snapshot**) is the desktop/engine layer's concern, versioned by the `embeddings_format_version` byte at offset 5. So, unlike `vault-prefs`, there is **no canonical-JSON inner shape** to pin: these vectors treat the plaintext as raw bytes and pin byte-transparency across the AEAD envelope. **The inner postcard snapshot is engine-opaque and is deliberately NOT pinned here** — only the envelope + AAD framing. (Same posture as `vault-search` / `vault-indices`.)

## Model-swap invalidation is NOT this layer's job

The model-identity tuple (`{model_hash, embedding_dim, chunker_version}`) rides **inside** the opaque payload (§ 9.17.2). A model / dimension / chunker change makes the desktop postcard decode fail (or compare-unequal), which the caller treats as fall-to-rebuild — the `vault.search` `extractor_version` precedent. The AAD stays vault-scoped like every sibling blob and does **NOT** bind the model identity. A porter must not attempt to invalidate on model change at this layer — the codec cannot see the model tuple.

## Two distinct load-bearing role strings

A porter must not conflate the two role strings this codec uses — they are different constants bound at different layers (and both are also distinct from the `search-*` / `vault-indices-*` families):

- **KEK (HKDF) role** `"ai-embeddings-key"` (17 ASCII bytes) — `ai_embeddings_key = DeriveSubkey(root_key, "ai-embeddings-key")` per § 3.2 + § 3.4. Pinned independently in the `hkdf-sha256` category.
- **AAD role** `"ai-embeddings-blob"` (18 ASCII bytes, length-prefixed) — the role inside `AAD_ai_embeddings_blob` per § 5.2 / § 9.17.3. Pinned independently in the `ctx-aead` category (`aad_canonical_bytes` case `aad-ai-embeddings-blob`) and decomposed in `aad-construction`.

This `embeddings` category exercises both end-to-end through the full encode/decode pipeline.

## Pipeline (as fixed by the reference implementation)

**Encrypt side** (`encode_embeddings_blob_raw`):

1. **Derive `ai_embeddings_key`** via the canonical helper: `derive_subkey(root_key, SubkeyRole::AiEmbeddingsKey)`. HKDF-SHA-256 with `info = "myne.v0.1 ai-embeddings-key"`, `salt = zero_32`. Output is 32 bytes, deterministic.
2. **Build the AAD** via `aad_ai_embeddings_blob(account_number_hash)`: 47-byte concatenation of `"myne.v0.1|"` (10 B) + protocol_version `0x01` (1 B) + length-prefixed UTF-8 role `"ai-embeddings-blob"` (2-B BE length `0x0012` + 18 ASCII bytes = 20 B) + `account_number_hash` (16 B) per spec § 9.17.3.
3. **AEAD-seal** the opaque plaintext under `ai_embeddings_key` + `nonce` + AAD using XChaCha20-Poly1305 with the CTX (Chan & Rogaway 2022) outer commitment per spec § 5.1. Output is `plaintext.len() + 32` bytes (CTX tag is 32 bytes).
4. **Assemble the outer envelope** per spec § 9.17.1 fixed-offset layout (total `34 + ciphertext_len` bytes):
   - `[0..4]` magic `MYR1`
   - `[4]` protocol_version `0x01`
   - `[5]` embeddings_format_version `0x01` (inner embedding-store serialization format — a future client bumps this so an old decoder rejects cleanly → fall to rebuild / re-embed)
   - `[6..30]` `nonce` (24 B)
   - `[30..34]` `ciphertext_len` (big-endian uint32, `plaintext.len() + 32`)
   - `[34..]` ciphertext + 32-byte CTX commitment tag

**Decrypt side** (`decode_embeddings_blob`): the inverse — parse header, validate magic / protocol_version / **embeddings_format_version** (all three checked **before** any decryption per the § 9.17.1 magic-byte-before-decrypt discipline), read `nonce` + `ciphertext_len` (reject `> 256 MiB + 32` cap — same cap as vault-search/vault-indices), rebuild AAD from `account_number_hash`, AEAD-decrypt. AEAD failure (wrong key / wrong AAD / tampered ciphertext) collapses to a single non-distinguishing `CannotDecrypt` per spec § 6.4.

The encode side is **pure and deterministic** given the caller-supplied `nonce` (no internal CSPRNG, no clock) — same inputs produce identical output bytes. Production callers use the `encode_embeddings_blob` convenience wrapper that pulls a nonce from `getrandom`; vectors pin a constant nonce for reproducibility.

## File layout

```
vectors/embeddings/
├── schema.md              ← this file
└── v0.1/
    └── cases.json         ← array of 12 KAT cases (encrypt + decrypt + AEAD-fail + structural parse-fail)
```

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/embeddings.json`](../../schema/v0.1/embeddings.json) (per-case shape with a 4-way `case_kind` discriminator: `encrypt`, `decrypt_ok`, `decrypt_aead_fail`, `parse_fail`). Run the repository's documented `ajv-cli validate` command (see the root `README.md`) to verify any `cases.json` against the canonical schema. The sub-schema is authoritative — this document covers only the domain-specific semantics.

Four case kinds carry different field shapes:

- **`encrypt`** — pins the encoder's byte output. Inputs: `input_root_key_hex` (32 B; the runner derives `ai_embeddings_key` from it), `input_account_number_hash_hex` (16 B), `input_plaintext_hex` (opaque, may be empty), `input_nonce_hex` (24 B). Optional `informational_ai_embeddings_key_hex` carries the pre-computed KEK for reviewer convenience. Output: `expected_embeddings_blob_hex` (the `MYR1` blob = 34-header + ciphertext + 32-CTX-tag).
- **`decrypt_ok`** — pins the decoder's successful round-trip. Inputs: `input_blob_hex` + correct `input_root_key_hex` + correct `input_account_number_hash_hex`. Output: `expected_plaintext_hex` (raw opaque bytes).
- **`decrypt_aead_fail`** — pins the **non-distinguishing** failure surface. Inputs: `input_blob_hex` + (intentionally wrong or tampered) inputs. Output: `expected_kind: "aead_fail"`. The decoder MUST collapse cross-vault (wrong `account_number_hash`), wrong-key, ciphertext-tamper, and CTX-tamper to a single indistinguishable failure (spec § 6.4). Fixtures carry the REAL key so a port that skips a check surfaces as a spurious `Ok` rather than a coincidental AEAD failure.
- **`parse_fail`** — pins a **distinguishable** pre-decryption structural reject. Inputs: `input_blob_hex` + `expected_error` (one of `magic_mismatch`, `protocol_version_mismatch`, `format_unsupported`, `header_too_short`, `ciphertext_truncated`, `ciphertext_len_overflow`). Unlike the AEAD-failure surface, these run **before** decryption and ARE distinguishable: a port must reject e.g. a future `embeddings_format_version` cleanly so the caller can fall to rebuild (re-embed) from the `.myn` files (§ 9.17.4).

## How to consume

A port runs each case based on `case_kind`:

- **`encrypt`**: derive `ai_embeddings_key` from `input_root_key_hex`, invoke the port's `encode_embeddings_blob_raw` equivalent with the supplied nonce + plaintext + account hash, hex-encode, compare byte-equal to `expected_embeddings_blob_hex`.
- **`decrypt_ok`**: derive the key, invoke the decoder, compare the recovered bytes to `expected_plaintext_hex`.
- **`decrypt_aead_fail`**: invoke the decoder, assert a non-distinguishing AEAD failure (the port's equivalent of `CannotDecrypt`). The port MUST NOT distinguish which arm fired.
- **`parse_fail`**: invoke the decoder, assert it returns the *specific* structural error named by `expected_error`, before any decryption. (Key/account-hash are irrelevant here — the reject precedes the AEAD stage.)

Any mismatch is a stop-the-world bug (per repo `README.md` "Cross-implementation discipline").

## Invariants the vectors pin

- **MYR1 magic + version + embeddings_format bytes.** Encoder output `[0..4]` = `MYR1`, `[4]` = `0x01`, `[5]` = `0x01`. Decoder rejects any other value at parse time before reaching AEAD verification — distinct, distinguishable errors (`parse_fail` cases). `MYR1 != MYS1 != MYI1` (separate families).
- **Header byte layout.** `[6..30]` = supplied `nonce` (24 B); `[30..34]` = `ciphertext_len` big-endian uint32; `[34..]` = ciphertext + 32-byte CTX tag.
- **Empty index round-trips.** An empty plaintext (no persisted index — the fall-to-rebuild / re-embed default) encodes to a 66-byte blob (34 header + 0 plaintext + 32 CTX tag) and decodes back to empty.
- **AAD binds account identity, not items_key lineage, and NOT model identity.** Cross-vault attack (`decrypt-cross-vault-account-hash`): a blob encoded under vault A's `account_number_hash` MUST fail to decrypt under vault B's. The `ai_embeddings_key` derives from `root_key` independent of the `itemsKey` lineage. The model-identity tuple is bound by the *payload*, not the AAD.
- **Tamper rejection.** Flipping a single bit in the ciphertext / CTX tag MUST cause a non-distinguishing AEAD failure (CTX commitment construction).
- **Non-distinguishing AEAD surface vs distinguishable structural surface.** The two `decrypt_aead_fail` cases (tamper + cross-vault) MUST be indistinguishable from each other; the six `parse_fail` cases MUST each surface their specific structural reject. This split is the spec § 6.4 boundary-policy discipline applied to the embeddings codec.
- **`embeddings_format_version` forward-migration.** The `parse-fail-format-unsupported` case pins that a v0.1 decoder rejects a higher `embeddings_format_version` cleanly (`FormatUnsupported`) so the caller falls to rebuild (re-embed) from the notes — the load-bearing forward hook (§ 9.17.4). NOTE: model-identity drift is handled inside the payload, not by bumping this byte.
- **`ciphertext_len` overflow guard.** The `parse-fail-ciphertext-len-overflow` case pins that an absurd `ciphertext_len` (`0xFFFFFFFF`, far above the 256 MiB + 32 cap) is rejected before any slice indexing / allocation — a port MUST bound-check before allocating.

## Regeneration

If the reference implementation changes (must not happen byte-stably without a major-version event), regenerate this vector via:

```bash
cargo test -p protocol --lib emit_embeddings_vectors_for_paste -- --ignored
cat /tmp/myne_embeddings_vectors.json
# copy the full JSON verbatim into cases.json
```

The regenerator test lives in `protocol/src/embeddings_blob.rs::tests::emit_embeddings_vectors_for_paste`. It is `#[ignore]`-gated so it never runs in default `cargo test` invocations.

## What this does NOT cover

- **Encode-only error modes.** `EmbeddingsBlobError::PlaintextTooLarge` (plaintext above the 256-MiB cap) and `EmbeddingsBlobError::RandomnessUnavailable` (OS CSPRNG failure in the non-`_raw` wrapper) are encode-side and cannot be expressed as a decode-a-blob case, so they are not vectored. The cap is pinned from the read direction by the `ciphertext-len-overflow` case instead.
- **The inner embedding-store serialization format (the postcard snapshot).** That is the desktop/engine layer's concern (client-opaque to this codec); a future `embeddings_format_version` bump changes it without touching this crypto framing. The model-identity tuple, the vector encoding, and the staleness manifest are all inside that opaque payload and are NOT pinned here.
- **Model-swap invalidation.** A model / dimension / chunker change is detected by the desktop layer via the payload-internal model-identity tuple (§ 9.17.2), NOT by this codec — so it is out of scope for these envelope vectors.
- **The Tauri-layer `read_or_rebuild` fall-back discipline** (absent file / truncated / AEAD-fail / unsupported-format all collapse to "re-embed from the `.myn` files" at the plugin boundary) — wrapper behavior is verified by `src-tauri` integration tests, not cross-implementation vectors.
- **Schema evolution within v0.1.** A future `embeddings_format_version` lands as a new case rather than mutating existing entries (immutable `vector_id`s).
