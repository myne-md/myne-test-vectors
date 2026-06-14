# `vault-indices` vector schema

Pins the byte mapping of `protocol::encode_vault_indices_blob_raw` + `protocol::decode_vault_indices_blob` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)). The codec emits and consumes `vault.indices` — the v0.1 vault-level singleton encrypted **structured-metadata index** blob (`MYI1` magic; MYND-271, design MYND-31) that lives next to `vault.meta` and carries the wikilink graph, the `#tag` index, the title→note resolution map, and the typed **property** records (every note's arbitrary frontmatter keys, each coerced to one Myne type per `markdown-dialect.md` § 3.6.4) in encrypted form.

**Reference implementation:** `protocol/src/vault_indices_blob.rs::{encode_vault_indices_blob_raw, decode_vault_indices_blob}`.
**Spec:** `docs/spec/protocol.md` § 9.8 (`vault.indices` outer envelope + engine-opaque inner plaintext + AAD + size cap + fall-to-rebuild semantic) + § 3.4 registry entry `"myne.v0.1 vault-indices-key"` (the KEK role) + § 5.2 / § 9.8.3 `AAD_vault_indices_blob` formula (the AAD role `"vault-indices-blob"`).

## Distinct blob family from `vault.search`

`MYI1` / `vault.indices` (structured metadata) and `MYS1` / `vault.search` (BM25 full-text) are **separate blob families** — one magic, one KEK role (`vault-indices-key` vs `search-index-key`), and one AAD role (`vault-indices-blob` vs `search-index-blob`) each. A blob of one family can never AEAD-verify under the other's key or AAD. (This category includes a unit-level pin — `search_key_cannot_open_indices_blob` — in the reference crate; the cross-family separation is load-bearing because the attachments catalog originally mis-claimed `MYI1`, `decisions.md` 2026-06-07.)

## Opaque inner plaintext (unlike `vault-prefs`)

`vault.indices` is **payload-agnostic**: the inner plaintext is an opaque byte string the desktop layer produces (the serialized structured index — wikilink edges + tag index + title map + typed property records, Build-B: TS types, Rust stores opaque). The protocol crate owns only the crypto framing — magic, version bytes, AEAD, AAD — while the index serialization is the desktop layer's concern, versioned by the `index_format_version` byte at offset 5. So, unlike `vault-prefs`, there is **no canonical-JSON inner shape** to pin: these vectors treat the plaintext as raw bytes and pin byte-transparency across the AEAD envelope. (Same posture as `vault-search` / `vault-workspace`; the four of them — prefs / search / workspace / indices — are the vault-singleton blob family.)

## Two distinct load-bearing role strings

A porter must not conflate the two role strings this codec uses — they are different constants bound at different layers (and both are also distinct from the `search-*` family):

- **KEK (HKDF) role** `"vault-indices-key"` (17 ASCII bytes) — `vault_indices_key = DeriveSubkey(root_key, "vault-indices-key")` per § 3.2 + § 3.4. Pinned independently in the `hkdf-sha256` category.
- **AAD role** `"vault-indices-blob"` (18 ASCII bytes, length-prefixed) — the role inside `AAD_vault_indices_blob` per § 5.2 / § 9.8.3. Pinned independently in the `ctx-aead` category (`aad_canonical_bytes` case `aad-vault-indices-blob`) and decomposed in `aad-construction`.

This `vault-indices` category exercises both end-to-end through the full encode/decode pipeline.

## Pipeline (as fixed by the reference implementation)

**Encrypt side** (`encode_vault_indices_blob_raw`):

1. **Derive `vault_indices_key`** via the canonical helper: `derive_subkey(root_key, SubkeyRole::VaultIndicesKey)`. HKDF-SHA-256 with `info = "myne.v0.1 vault-indices-key"`, `salt = zero_32`. Output is 32 bytes, deterministic.
2. **Build the AAD** via `aad_vault_indices_blob(account_number_hash)`: 47-byte concatenation of `"myne.v0.1|"` (10 B) + protocol_version `0x01` (1 B) + length-prefixed UTF-8 role `"vault-indices-blob"` (2-B BE length `0x0012` + 18 ASCII bytes = 20 B) + `account_number_hash` (16 B) per spec § 9.8.3.
3. **AEAD-seal** the opaque plaintext under `vault_indices_key` + `nonce` + AAD using XChaCha20-Poly1305 with the CTX (Chan & Rogaway 2022) outer commitment per spec § 5.1. Output is `plaintext.len() + 32` bytes (CTX tag is 32 bytes).
4. **Assemble the outer envelope** per spec § 9.8.1 fixed-offset layout (total `34 + ciphertext_len` bytes):
   - `[0..4]` magic `MYI1`
   - `[4]` protocol_version `0x01`
   - `[5]` index_format_version `0x01` (inner structured-index serialization format — a future client bumps this so an old decoder rejects cleanly → fall to rebuild)
   - `[6..30]` `nonce` (24 B)
   - `[30..34]` `ciphertext_len` (big-endian uint32, `plaintext.len() + 32`)
   - `[34..]` ciphertext + 32-byte CTX commitment tag

**Decrypt side** (`decode_vault_indices_blob`): the inverse — parse header, validate magic / protocol_version / **index_format_version** (all three checked **before** any decryption per the § 9.8.1 magic-byte-before-decrypt discipline), read `nonce` + `ciphertext_len` (reject `> 256 MiB + 32` cap — same cap as vault-search, larger than vault-workspace's 1 MiB), rebuild AAD from `account_number_hash`, AEAD-decrypt. AEAD failure (wrong key / wrong AAD / tampered ciphertext) collapses to a single non-distinguishing `CannotDecrypt` per spec § 6.4.

The encode side is **pure and deterministic** given the caller-supplied `nonce` (no internal CSPRNG, no clock) — same inputs produce identical output bytes. Production callers use the `encode_vault_indices_blob` convenience wrapper that pulls a nonce from `getrandom`; vectors pin a constant nonce for reproducibility.

## File layout

```
vectors/vault-indices/
├── schema.md              ← this file
└── v0.1/
    └── cases.json         ← array of 12 KAT cases (encrypt + decrypt + AEAD-fail + structural parse-fail)
```

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/vault-indices.json`](../../schema/v0.1/vault-indices.json) (per-case shape with a 4-way `case_kind` discriminator: `encrypt`, `decrypt_ok`, `decrypt_aead_fail`, `parse_fail`). Run the repository's documented `ajv-cli validate` command (see the root `README.md`) to verify any `cases.json` against the canonical schema. The sub-schema is authoritative — this document covers only the domain-specific semantics.

Four case kinds carry different field shapes:

- **`encrypt`** — pins the encoder's byte output. Inputs: `input_root_key_hex` (32 B; the runner derives `vault_indices_key` from it), `input_account_number_hash_hex` (16 B), `input_plaintext_hex` (opaque, may be empty), `input_nonce_hex` (24 B). Optional `informational_vault_indices_key_hex` carries the pre-computed KEK for reviewer convenience. Output: `expected_vault_indices_blob_hex` (the `MYI1` blob = 34-header + ciphertext + 32-CTX-tag).
- **`decrypt_ok`** — pins the decoder's successful round-trip. Inputs: `input_blob_hex` + correct `input_root_key_hex` + correct `input_account_number_hash_hex`. Output: `expected_plaintext_hex` (raw opaque bytes).
- **`decrypt_aead_fail`** — pins the **non-distinguishing** failure surface. Inputs: `input_blob_hex` + (intentionally wrong or tampered) inputs. Output: `expected_kind: "aead_fail"`. The decoder MUST collapse cross-vault (wrong `account_number_hash`), wrong-key, ciphertext-tamper, and CTX-tamper to a single indistinguishable failure (spec § 6.4). Fixtures carry the REAL key so a port that skips a check surfaces as a spurious `Ok` rather than a coincidental AEAD failure.
- **`parse_fail`** — pins a **distinguishable** pre-decryption structural reject. Inputs: `input_blob_hex` + `expected_error` (one of `magic_mismatch`, `protocol_version_mismatch`, `format_unsupported`, `header_too_short`, `ciphertext_truncated`, `ciphertext_len_overflow`). Unlike the AEAD-failure surface, these run **before** decryption and ARE distinguishable: a port must reject e.g. a future `index_format_version` cleanly so the caller can fall to rebuild from the `.myn` files (§ 9.8.4).

## How to consume

A port runs each case based on `case_kind`:

- **`encrypt`**: derive `vault_indices_key` from `input_root_key_hex`, invoke the port's `encode_vault_indices_blob_raw` equivalent with the supplied nonce + plaintext + account hash, hex-encode, compare byte-equal to `expected_vault_indices_blob_hex`.
- **`decrypt_ok`**: derive the key, invoke the decoder, compare the recovered bytes to `expected_plaintext_hex`.
- **`decrypt_aead_fail`**: invoke the decoder, assert a non-distinguishing AEAD failure (the port's equivalent of `CannotDecrypt`). The port MUST NOT distinguish which arm fired.
- **`parse_fail`**: invoke the decoder, assert it returns the *specific* structural error named by `expected_error`, before any decryption. (Key/account-hash are irrelevant here — the reject precedes the AEAD stage.)

Any mismatch is a stop-the-world bug (per repo `README.md` "Cross-implementation discipline").

## Invariants the vectors pin

- **MYI1 magic + version + index_format bytes.** Encoder output `[0..4]` = `MYI1`, `[4]` = `0x01`, `[5]` = `0x01`. Decoder rejects any other value at parse time before reaching AEAD verification — distinct, distinguishable errors (`parse_fail` cases). `MYI1 != MYS1` (separate family).
- **Header byte layout.** `[6..30]` = supplied `nonce` (24 B); `[30..34]` = `ciphertext_len` big-endian uint32; `[34..]` = ciphertext + 32-byte CTX tag.
- **Empty index round-trips.** An empty plaintext (no persisted index — the fall-to-rebuild default) encodes to a 66-byte blob (34 header + 0 plaintext + 32 CTX tag) and decodes back to empty.
- **AAD binds account identity, not items_key lineage.** Cross-vault attack (`decrypt-cross-vault-account-hash`): a blob encoded under vault A's `account_number_hash` MUST fail to decrypt under vault B's. The `vault_indices_key` derives from `root_key` independent of the `itemsKey` lineage.
- **Tamper rejection.** Flipping a single bit in the ciphertext / CTX tag MUST cause a non-distinguishing AEAD failure (CTX commitment construction).
- **Non-distinguishing AEAD surface vs distinguishable structural surface.** The two `decrypt_aead_fail` cases (tamper + cross-vault) MUST be indistinguishable from each other; the six `parse_fail` cases MUST each surface their specific structural reject. This split is the spec § 6.4 boundary-policy discipline applied to the vault-indices codec.
- **`index_format_version` forward-migration.** The `parse-fail-format-unsupported` case pins that a v0.1 decoder rejects a higher `index_format_version` cleanly (`FormatUnsupported`) so the caller falls to rebuild from the notes — the load-bearing forward hook (§ 9.8.4).
- **`ciphertext_len` overflow guard.** The `parse-fail-ciphertext-len-overflow` case pins that an absurd `ciphertext_len` (`0xFFFFFFFF`, far above the 256 MiB + 32 cap) is rejected before any slice indexing / allocation — a port MUST bound-check before allocating.

## Regeneration

If the reference implementation changes (must not happen byte-stably without a major-version event), regenerate this vector via:

```bash
cargo test -p protocol --lib emit_vault_indices_vectors_for_paste -- --ignored
cat /tmp/myne_vault_indices_vectors.json
# copy the full JSON verbatim into cases.json
```

The regenerator test lives in `protocol/src/vault_indices_blob.rs::tests::emit_vault_indices_vectors_for_paste`. It is `#[ignore]`-gated so it never runs in default `cargo test` invocations.

## What this does NOT cover

- **Encode-only error modes.** `VaultIndicesBlobError::PlaintextTooLarge` (plaintext above the 256-MiB cap) and `VaultIndicesBlobError::RandomnessUnavailable` (OS CSPRNG failure in the non-`_raw` wrapper) are encode-side and cannot be expressed as a decode-a-blob case, so they are not vectored. The cap is pinned from the read direction by the `ciphertext-len-overflow` case instead.
- **The inner structured-index serialization format.** That is the desktop layer's concern (client-opaque to this codec); a future `index_format_version` bump changes it without touching this crypto framing. The value-typing of properties that *feeds* the index is pinned separately by the `property-coercion` category.
- **The Tauri-layer `read_or_rebuild` fall-back discipline** (absent file / truncated / AEAD-fail / unsupported-format all collapse to "rebuild from the `.myn` files" at the plugin boundary) — wrapper behavior is verified by `src-tauri` integration tests, not cross-implementation vectors.
- **Schema evolution within v0.1.** A future `index_format_version` lands as a new case rather than mutating existing entries (immutable `vector_id`s).
