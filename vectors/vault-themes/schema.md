# `vault-themes` vector schema

Pins the byte mapping of `protocol::encode_themes_blob_raw` + `protocol::decode_themes_blob` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)). The codec emits and consumes `vault.themes` — the v0.1 vault-level singleton encrypted **custom-themes** blob (`MYT1` magic; MYND-486) that lives next to `vault.meta` and carries the user's `{theme_name → palette}` store in encrypted form.

**Reference implementation:** `protocol/src/themes_blob.rs::{encode_themes_blob_raw, decode_themes_blob}`.
**Spec:** `docs/spec/protocol.md` § 9.21 (`vault.themes` outer envelope + opaque inner plaintext + AAD + size cap + fall-to-empty semantic) + § 3.4 registry entry `"myne.v0.1 custom-themes-key"` (the KEK role) + § 5.2 / § 9.21.3 `AAD_custom_themes_blob` formula (the AAD role `"custom-themes-blob"`).

Before MYND-486 the palettes were device-local **plaintext** at `app_config_dir/themes/<name>.json`. Promoting them to vault content makes them encrypted, synced, and backed up (`decisions.md` 2026-07-16 supersedes the 2026-06-16 storage-split clause).

## Opaque inner plaintext (unlike `vault-prefs`)

`vault.themes` is **payload-agnostic**: the inner plaintext is an opaque byte string the client layer produces (at `themes_format_version` `0x01`, a deterministic-JSON `{name → {palette, created_at, updated_at, deleted_at?}}` map whose per-name wall clocks drive the Phase 2 sync merge). The protocol crate owns only the crypto framing — magic, version bytes, AEAD, AAD — while the map serialization is the client layer's concern, versioned by the `themes_format_version` byte at offset 5. So, unlike `vault-prefs`, there is **no canonical-JSON inner shape** to pin: these vectors treat the plaintext as raw bytes and pin byte-transparency across the AEAD envelope. (Same posture as `vault-search` / `vault-workspace` / `vault-indices` / `vault-dictionary` — the vault-singleton blob family, one magic + one KEK + one AAD role each.)

## Two distinct load-bearing role strings

A porter must not conflate the two role strings this codec uses — they are different constants bound at different layers:

- **KEK (HKDF) role** `"custom-themes-key"` (17 ASCII bytes) — `custom_themes_key = DeriveSubkey(root_key, "custom-themes-key")` per § 3.2 + § 3.4. Pinned independently in the `hkdf-sha256` category.
- **AAD role** `"custom-themes-blob"` (18 ASCII bytes, length-prefixed) — the role inside `AAD_custom_themes_blob` per § 5.2 / § 9.21.3. Decomposed independently in the `aad-construction` category.

This `vault-themes` category exercises both end-to-end through the full encode/decode pipeline.

A **third** themes-related registry entry exists and is *not* exercised here: `"themes-sync-id"` (§ 3.4) derives the reserved 16-byte sync blob-id of `vault.themes` for Phase 2 sync (`sync-protocol.md` § 10.2.2). It is an **identifier** derivation, not a KEK — the themes KEK stays `custom-themes-key`. It is pinned in the `hkdf-sha256` category alongside its `catalog-sync-id` / `prefs-sync-id` siblings.

## Pipeline (as fixed by the reference implementation)

**Encrypt side** (`encode_themes_blob_raw`):

1. **Derive `custom_themes_key`** via the canonical helper: `derive_subkey(root_key, SubkeyRole::CustomThemesKey)`. HKDF-SHA-256 with `info = "myne.v0.1 custom-themes-key"`, `salt = zero_32`. Output is 32 bytes, deterministic.
2. **Build the AAD** via `aad_custom_themes_blob(account_number_hash)`: 47-byte concatenation of `"myne.v0.1|"` (10 B) + protocol_version `0x01` (1 B) + length-prefixed UTF-8 role `"custom-themes-blob"` (2-B BE length `0x0012` + 18 ASCII bytes = 20 B) + `account_number_hash` (16 B) per spec § 9.21.3.
3. **AEAD-seal** the opaque plaintext under `custom_themes_key` + `nonce` + AAD using XChaCha20-Poly1305 with the CTX (Chan & Rogaway 2022) outer commitment per spec § 5.1. Output is `plaintext.len() + 32` bytes (CTX tag is 32 bytes).
4. **Assemble the outer envelope** per spec § 9.21.1 fixed-offset layout (total `34 + ciphertext_len` bytes):
   - `[0..4]` magic `MYT1`
   - `[4]` protocol_version `0x01`
   - `[5]` themes_format_version `0x01` (inner map serialization format — a future client bumps this so an old decoder rejects cleanly → fall to the empty theme store)
   - `[6..30]` `nonce` (24 B)
   - `[30..34]` `ciphertext_len` (big-endian uint32, `plaintext.len() + 32`)
   - `[34..]` ciphertext + 32-byte CTX commitment tag

**Decrypt side** (`decode_themes_blob`): the inverse — parse header, validate magic / protocol_version / **themes_format_version** (all three checked **before** any decryption per the § 9.21.1 magic-byte-before-decrypt discipline), read `nonce` + `ciphertext_len` (reject `> 4 MiB + 32` cap), rebuild AAD from `account_number_hash`, AEAD-decrypt. AEAD failure (wrong key / wrong AAD / tampered ciphertext) collapses to a single non-distinguishing `CannotDecrypt` per spec § 6.4.

The encode side is **pure and deterministic** given the caller-supplied `nonce` (no internal CSPRNG, no clock) — same inputs produce identical output bytes. Production callers use the `encode_themes_blob` convenience wrapper that pulls a nonce from `getrandom`; vectors pin a constant nonce for reproducibility.

## File layout

```
vectors/vault-themes/
├── schema.md              ← this file
├── v0.1/
│   └── cases.json         ← array of 12 KAT cases (encrypt + decrypt + AEAD-fail + structural parse-fail)
└── v0.2/
    └── cases.json         ← 2 cases pinning the MYT5 envelope (see "The v0.2 side" below)
```

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/vault-themes.json`](../../schema/v0.1/vault-themes.json) (per-case shape with a 4-way `case_kind` discriminator: `encrypt`, `decrypt_ok`, `decrypt_aead_fail`, `parse_fail`). Run the repository's documented `ajv-cli validate` command (see the root `README.md`) to verify any `cases.json` against the canonical schema. The sub-schema is authoritative — this document covers only the domain-specific semantics.

Four case kinds carry different field shapes:

- **`encrypt`** — pins the encoder's byte output. Inputs: `input_root_key_hex` (32 B; the runner derives `custom_themes_key` from it), `input_account_number_hash_hex` (16 B), `input_plaintext_hex` (opaque, may be empty), `input_nonce_hex` (24 B). Optional `informational_custom_themes_key_hex` carries the pre-computed KEK for reviewer convenience. Output: `expected_vault_themes_blob_hex` (the `MYT1` blob = 34-header + ciphertext + 32-CTX-tag).
- **`decrypt_ok`** — pins the decoder's successful round-trip. Inputs: `input_blob_hex` + correct `input_root_key_hex` + correct `input_account_number_hash_hex`. Output: `expected_plaintext_hex` (raw opaque bytes).
- **`decrypt_aead_fail`** — pins the **non-distinguishing** failure surface. Inputs: `input_blob_hex` + (intentionally wrong or tampered) inputs. Output: `expected_kind: "aead_fail"`. The decoder MUST collapse cross-vault (wrong `account_number_hash`), wrong-key, ciphertext-tamper, and CTX-tamper to a single indistinguishable failure (spec § 6.4). Fixtures carry the REAL key so a port that skips a check surfaces as a spurious `Ok` rather than a coincidental AEAD failure.
- **`parse_fail`** — pins a **distinguishable** pre-decryption structural reject. Inputs: `input_blob_hex` + `expected_error` (one of `magic_mismatch`, `protocol_version_mismatch`, `format_unsupported`, `header_too_short`, `ciphertext_truncated`, `ciphertext_len_overflow`). Unlike the AEAD-failure surface, these run **before** decryption and ARE distinguishable: a port must reject e.g. a future `themes_format_version` cleanly so the caller can fall back to the empty theme store (§ 9.21.4). The Rust reference names the format-version reject `ThemesBlobError::ThemesFormatUnsupported`; the vector string stays `format_unsupported` for cross-family consistency.

## How to consume

A port runs each case based on `case_kind`:

- **`encrypt`**: derive `custom_themes_key` from `input_root_key_hex`, invoke the port's `encode_themes_blob_raw` equivalent with the supplied nonce + plaintext + account hash, hex-encode, compare byte-equal to `expected_vault_themes_blob_hex`.
- **`decrypt_ok`**: derive the key, invoke the decoder, compare the recovered bytes to `expected_plaintext_hex`.
- **`decrypt_aead_fail`**: invoke the decoder, assert a non-distinguishing AEAD failure (the port's equivalent of `CannotDecrypt`). The port MUST NOT distinguish which arm fired.
- **`parse_fail`**: invoke the decoder, assert it returns the *specific* structural error named by `expected_error`, before any decryption. (Key/account-hash are irrelevant here — the reject precedes the AEAD stage.)

Any mismatch is a stop-the-world bug (per repo `README.md` "Cross-implementation discipline").

## Invariants the vectors pin

- **MYT1 magic + version + themes_format bytes.** Encoder output `[0..4]` = `MYT1`, `[4]` = `0x01`, `[5]` = `0x01`. Decoder rejects any other value at parse time before reaching AEAD verification — distinct, distinguishable errors (`parse_fail` cases). `MYT1` is its own blob family (one magic; `MYT1` ≠ `MYD1` ≠ `MYS1`).
- **Header byte layout.** `[6..30]` = supplied `nonce` (24 B); `[30..34]` = `ciphertext_len` big-endian uint32; `[34..]` = ciphertext + 32-byte CTX tag.
- **Empty theme store round-trips.** An empty plaintext (the fresh-vault / fall-to-empty default) encodes to a 66-byte blob (34 header + 0 plaintext + 32 CTX tag) and decodes back to empty.
- **AAD binds account identity, not items_key lineage.** Cross-vault attack (`decrypt-cross-vault-account-hash`): a blob encoded under vault A's `account_number_hash` MUST fail to decrypt under vault B's. The `custom_themes_key` derives from `root_key` independent of the `itemsKey` lineage, so the themes store survives an items-key rotation.
- **Tamper rejection.** Flipping a single bit in the ciphertext / CTX tag MUST cause a non-distinguishing AEAD failure (CTX commitment construction).
- **Non-distinguishing AEAD surface vs distinguishable structural surface.** The two `decrypt_aead_fail` cases (tamper + cross-vault) MUST be indistinguishable from each other; the six `parse_fail` cases MUST each surface their specific structural reject. This split is the spec § 6.4 boundary-policy discipline applied to the vault-themes codec.
- **`themes_format_version` forward-migration.** The `parse-fail-format-unsupported` case pins that a v0.1 decoder rejects a higher `themes_format_version` cleanly (`ThemesFormatUnsupported`) so the caller falls back to the empty theme store — the load-bearing forward hook (§ 9.21.4).
- **`ciphertext_len` overflow guard.** The `parse-fail-ciphertext-len-overflow` case pins that an absurd `ciphertext_len` (`0xFFFFFFFF`, far above the 4 MiB + 32 cap) is rejected before any slice indexing / allocation — a port MUST bound-check before allocating.

## Regeneration

If the reference implementation changes (must not happen byte-stably without a major-version event), regenerate this vector via:

```bash
cargo test -p protocol --lib emit_themes_blob_vectors_for_paste -- --ignored
cat /tmp/myne_vault_themes_vectors.json
# copy the full JSON verbatim into cases.json
```

The regenerator test lives in `protocol/src/themes_blob.rs::tests::emit_themes_blob_vectors_for_paste`. It is `#[ignore]`-gated so it never runs in default `cargo test` invocations.

## The v0.2 side (`v0.2/cases.json`)

`v0.2/` pins the `MYT5` envelope — the v0.2 side of the §10.4.1 md.6
`protocol_version` split (myne-md/myne#441), and the KEK-envelope archetype the
other single-KEK families share byte-for-byte.

**One detail here reads like a bug and is not.** The header's
`themes_format_version` byte stays `0x01` even though the family is v0.2. That
byte versions the *inner payload format*, not the protocol major, and it is not
AEAD-bound — the domain separation is carried entirely by the magic (`MYT5` vs
`MYT1`), the AAD prefix (`"myne.v0.2|"`, via `aad_singleton_blob_v0_2` with the
`"custom-themes-blob"` role) and the KEK's HKDF context prefix (`"myne.v0.2 "`).
A port that "corrects" this byte to `0x02` produces a blob the reference decoder
rejects. The sub-schema pins it as a `const` for that reason.

The second case kind, **`decode_fail_v0_1`**, is the mixed-fleet direction: an
un-migrated client meeting a migrated vault's theme store must not open it.
Either refusal is correct — `cannot_decrypt` (the AEAD simply does not verify,
since the tag was computed under the v0.2 AAD prefix; per §6.4 that failure stays
non-distinguishing) or `magic_mismatch` (an implementation that checks the magic
before attempting the AEAD). What a port may **not** do is return a plaintext.

Schema: [`schema/v0.2.json`](../../schema/v0.2.json) +
[`schema/v0.2/vault-themes.json`](../../schema/v0.2/vault-themes.json).
Generator: `emit_vault_themes_v0_2_vectors_for_paste` (`protocol/src/themes_blob.rs`).
Runner: `vault_themes_v0_2_vectors` (`protocol/tests/vectors.rs`).

## What this does NOT cover

- **Encode-only error modes.** `ThemesBlobError::PlaintextTooLarge` (plaintext above the 4-MiB cap) and `ThemesBlobError::RandomnessUnavailable` (OS CSPRNG failure in the non-`_raw` wrapper) are encode-side and cannot be expressed as a decode-a-blob case, so they are not vectored. The cap is pinned from the read direction by the `ciphertext-len-overflow` case instead.
- **The inner `{name → palette}` serialization format.** That is the client layer's concern (client-opaque to this codec); a future `themes_format_version` bump changes it without touching this crypto framing. The per-name wall clocks that drive the sync merge live inside those opaque bytes.
- **The Phase 2 sync blob-id derivation** (`"themes-sync-id"`, § 3.4 / `sync-protocol.md` § 10.2.2) — an identifier derivation pinned in `hkdf-sha256`, not part of this codec's framing.
- **The client-layer fall-back discipline** (absent file / truncated / AEAD-fail / unsupported-format all collapse to "the empty theme store") — wrapper behavior is verified by `myne-core` / `src-tauri` integration tests, not cross-implementation vectors.
- **Schema evolution within v0.1.** A future `themes_format_version` lands as a new case rather than mutating existing entries (immutable `vector_id`s).
