# `chats-blob` vector schema

Pins the byte mapping of `protocol::encode_chats_blob_raw` + `protocol::decode_chats_blob` + `protocol::aad_chats_blob` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)). The codec emits and consumes one `chats/<conversation-id-hex>` file — the v0.1 per-conversation encrypted **turn-list payload** (`MYJ1` magic, myne-md/myne#675).

**Reference implementation:** `protocol/src/chats_blob.rs::{encode_chats_blob_raw, decode_chats_blob, aad_chats_blob}`.
**Spec:** `docs/spec/protocol.md` § 9.24 (`MYJ1` envelope + turn-list inner shape + caps) + § 5.2 (AAD formula, role `"chats-blob"`) + § 3.4 registry entries `"chats-key"` (the family KEK role).

## Turn-list inner plaintext

The inner plaintext is the shipped writer's turn-list payload JSON (`{turns, title_derived_from}` — key order not significant, writers emit `turns` first; `docs/spec/protocol.md` §9.24.2); the codec treats it as opaque bytes and pins them. A settled agent turn's frozen proposal (anchors + replacements verbatim) rides inside the payload — the `encrypt-proposal` case pins exactly that shape through the codec.

## The per-entry binding (the MYE1 archetype)

- **KEK (HKDF) role** `"chats-key"` (9 ASCII bytes) — ONE KEK serves every conversation file: `chats_key = DeriveSubkey(root_key, "chats-key")` per § 3.2 + § 3.4. Pinned independently in the `hkdf-sha256` category.
- **AAD role** `"chats-blob"` (10 ASCII bytes, length-prefixed) — the role inside `aad_chats_blob`: `"myne.v0.1|"` (10 B) + protocol_version `0x01` (1 B) + 2-B BE length `0x000a` + role (10 B) + `conversation_id` (16 B) + `account_number_hash` (16 B) = 55 bytes. Binding the per-entry id is what makes one KEK safe across all files: a blob sealed for conversation A fails to verify as B. Pinned by the `aad-construction` case, and by the `decrypt-fail-cross-conversation` case here.

## Pipeline (as fixed by the reference implementation)

**Encrypt side** (`encode_chats_blob_raw`): derive `chats_key`, take the payload bytes as opaque (16-MiB plaintext cap — refused, never truncated, § 9.24.4), AEAD-seal under `chats_key` + `nonce` + the 55-byte AAD, assemble the outer envelope (total `34 + ciphertext_len` bytes): `[0..4]` magic `MYJ1`, `[4]` protocol_version `0x01`, `[5]` format_version `0x01`, `[6..30]` nonce (24 B), `[30..34]` `ciphertext_len` (big-endian uint32, `plaintext.len() + 32`), `[34..]` ciphertext + 32-byte CTX commitment tag.

**Decrypt side** (`decode_chats_blob`): parse + validate the header before any decryption, rebuild the AAD from `account_number_hash` + `conversation_id`, AEAD-decrypt. AEAD failure collapses to the single non-distinguishing `CannotDecrypt` per § 6.4.

## The v0.2 side

`vectors/chats-blob/v0.2/` pins the migrated envelope (`MYJ5`, v0.2 KEK via `derive_subkey_versioned` + v0.2 AAD): one `encrypt` case plus the `decode_fail_v0_1` cross-generation refusal. Unlike the catalog/themes singletons, the conversation-file header version byte IS the generation (`0x02` here) — pinned as a `const` precisely because the two families differ.

## File layout

```
vectors/chats-blob/
├── schema.md              ← this file
├── v0.1/
│   └── cases.json         ← array of 13 KAT cases (encrypt + decrypt + AEAD-fail + parse-fail)
└── v0.2/
    └── cases.json         ← MYJ5 envelope + cross-generation refusal
```

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/chats-blob.json`](../../schema/v0.1/chats-blob.json) (per-case shape with a 4-way `case_kind` discriminator) plus the v0.2 pair. The sub-schema is authoritative — this document covers only the domain-specific semantics.

- **`encrypt`** (`encrypt-empty`, `encrypt-populated`, `encrypt-proposal`) — pins the encoder's byte output. The empty case is the create-state payload (no turns yet); the proposal case pins a frozen agent proposal through the codec. Inputs: `input_root_key_hex` (the runner derives `chats_key`), `input_account_number_hash_hex`, `input_conversation_id_hex`, `input_plaintext_hex`, `input_blob_nonce_hex`; optional `informational_chats_key_hex`. Output: `expected_blob_hex`.
- **`decrypt_ok`** (`decrypt-ok`) — pins the decoder's success path.
- **`decrypt_aead_fail`** (`decrypt-fail-tamper`, `decrypt-fail-cross-vault`, `decrypt-fail-cross-conversation`) — non-distinguishing failure surface (`expected_kind: "aead_fail"`). The third case is the load-bearing one for this family: same vault, wrong conversation id.
- **`parse_fail`** (`parse-fail-*`) — the **distinguishable** reject surface, one case per `ChatsBlobError` variant reachable from a decode: the six pre-AEAD structural rejects (`header_too_short`, `magic_mismatch`, `protocol_version_mismatch`, `format_unsupported`, `ciphertext_len_overflow`, `ciphertext_truncated`). `expected_error` names the variant. The cases carry the REAL `root_key` deliberately: header bytes are not AAD-bound, so a port that skips a structural check decrypts the tampered blob successfully and returns Ok — which the runner catches.
