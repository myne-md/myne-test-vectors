# `chats-catalog` vector schema

Pins the byte mapping of `protocol::encode_chats_catalog_blob_raw` + `protocol::decode_chats_catalog_blob` + `protocol::aad_chats_catalog` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)). The codec emits and consumes `vault.chats` — the v0.1 vault-level singleton encrypted **conversations catalog** (`MYM1` magic, myne-md/myne#675) that lives next to `vault.meta` / `vault.prefs` / `vault.search` and maps conversation-ids to their roster rows.

**Reference implementation:** `protocol/src/chats_catalog_blob.rs::{encode_chats_catalog_blob_raw, decode_chats_catalog_blob, aad_chats_catalog}`.
**Spec:** `docs/spec/protocol.md` § 9.23 (`MYM1` envelope + roster inner shape + report-don't-empty + `read_or_empty` collapse) + § 5.2 (AAD formula, role `"chats-catalog"`) + § 3.4 registry entries `"chats-catalog-key"` (the KEK role) and `"chats-catalog-sync-id"` (the Phase-2 sync blob-id role).

## Canonical roster inner plaintext (like `attachments-catalog`)

The inner plaintext is the client layer's canonical roster JSON: an object mapping conversation-id hex → `{created_at, deleted_at?, title, updated_at, version?}`. Canonical form fixed by the writer:

- Top-level keys (conversation-ids) in lexicographic order; per-entry keys in alphabetical order `created_at` < `deleted_at` < `title` < `updated_at` < `version`, with `version: 0` omitted and `deleted_at` present only on tombstones.
- No insignificant whitespace; empty roster = `{}` (2 bytes, `7b7d`).

Unlike `attachments-catalog` there is no forward-compat unknown-field tolerance case: the catalog is **load-bearing user content**, so an unreadable catalog is REPORTED (§ 9.23.3 report-don't-empty — never collapsed to "no conversations"), and tolerance of unknown fields would silently drop roster rows a newer client wrote.

## Two distinct load-bearing role strings, plus the sync identifier

- **KEK (HKDF) role** `"chats-catalog-key"` (17 ASCII bytes) — `catalog_key = DeriveSubkey(root_key, "chats-catalog-key")` per § 3.2 + § 3.4. Pinned independently in the `hkdf-sha256` category.
- **Sync-id (HKDF) role** `"chats-catalog-sync-id"` (21 ASCII bytes) — the Phase-2 sync blob-id of `vault.chats` (first 16 bytes of the output), an identifier derivation like `catalog-sync-id`, NOT a KEK. Pinned independently in the `hkdf-sha256` category.
- **AAD role** `"chats-catalog"` (13 ASCII bytes, length-prefixed) — the role inside `aad_chats_catalog`: `"myne.v0.1|"` (10 B) + protocol_version `0x01` (1 B) + 2-B BE length `0x000d` + role (13 B) + `account_number_hash` (16 B) = 42 bytes. Pinned by the `aad-construction` case.

## Pipeline (as fixed by the reference implementation)

**Encrypt side** (`encode_chats_catalog_blob_raw`):

1. **Derive `catalog_key`** via the canonical helper: `derive_subkey(root_key, SubkeyRole::ChatsCatalogKey)`.
2. **Take the roster plaintext as opaque bytes** (the client layer owns the canonical form above; 4-MiB plaintext cap).
3. **AEAD-seal** under `catalog_key` + `nonce` + the 42-byte AAD (XChaCha20-Poly1305 + 32-byte CTX commitment per § 5.1).
4. **Assemble the outer envelope** (total `34 + ciphertext_len` bytes):
   - `[0..4]` magic `MYM1`
   - `[4]` protocol_version `0x01`
   - `[5]` format_version `0x01`
   - `[6..30]` nonce (24 B)
   - `[30..34]` `ciphertext_len` (big-endian uint32, `plaintext.len() + 32`)
   - `[34..]` ciphertext + 32-byte CTX commitment tag

**Decrypt side** (`decode_chats_catalog_blob`): parse + validate the header before any decryption, rebuild the AAD from `account_number_hash`, AEAD-decrypt. AEAD failure collapses to the single non-distinguishing `CannotDecrypt` per § 6.4.

## The v0.2 side

`vectors/chats-catalog/v0.2/` pins the migrated envelope (`MYM5`, v0.2 KEK via `derive_subkey_versioned` + v0.2 AAD): one `encrypt` case plus the `decode_fail_v0_1` cross-generation refusal. The header version byte stays `0x01` (the envelope version, not the generation), like the themes singleton family.

## File layout

```
vectors/chats-catalog/
├── schema.md              ← this file
├── v0.1/
│   └── cases.json         ← array of 13 KAT cases (encrypt + decrypt + AEAD-fail + parse-fail)
└── v0.2/
    └── cases.json         ← MYM5 envelope + cross-generation refusal
```

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/chats-catalog.json`](../../schema/v0.1/chats-catalog.json) (per-case shape with a 4-way `case_kind` discriminator) plus the v0.2 pair. The sub-schema is authoritative — this document covers only the domain-specific semantics.

- **`encrypt`** (`encrypt-empty`, `encrypt-populated`, `encrypt-tombstoned`) — pins the encoder's byte output over the roster plaintext. The tombstoned case is the **enrollment seed**: a roster as an enrolling device receives it (one live conversation beside a tombstone), pinning that the seed is wholesale — the tombstone keeps every field plus `deleted_at`, so the conversation does not resurrect on the new device (`sync-protocol.md` enrollment seed). Inputs: `input_root_key_hex` (the runner derives `catalog_key`), `input_account_number_hash_hex`, `input_plaintext_canonical_utf8_hex`, `input_catalog_nonce_hex`; optional `informational_catalog_key_hex`. Output: `expected_catalog_blob_hex`.
- **`decrypt_ok`** (`decrypt-ok`, `decrypt-ok-tombstoned`) — pins the decoder's success path. `expected_plaintext_canonical_utf8_hex` is the sealed plaintext bytes; the tombstoned case pins the tombstone surviving intact.
- **`decrypt_aead_fail`** (`decrypt-fail-tamper`, `decrypt-fail-cross-vault`) — non-distinguishing failure surface (`expected_kind: "aead_fail"`).
- **`parse_fail`** (`parse-fail-*`) — the **distinguishable** reject surface, one case per `ChatsCatalogError` variant reachable from a decode: the six pre-AEAD structural rejects (`header_too_short`, `magic_mismatch`, `protocol_version_mismatch`, `format_unsupported`, `ciphertext_len_overflow`, `ciphertext_truncated`). `expected_error` names the variant. The cases carry the REAL `root_key` deliberately: header bytes are not AAD-bound, so a port that skips a structural check decrypts the tampered blob successfully and returns Ok — which the runner catches.
