# `attachment-text` vector schema

Pins the byte mapping of `protocol::encode_attachment_text_blob_raw` + `protocol::decode_attachment_text_blob` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)). The codec emits and consumes a **per-attachment extracted-text cache blob** — the v0.1 `MYE1` blob (MYND-328) stored at `attachments-text/<attachment-id-hex>`, mirroring the `attachments/<attachment-id-hex>` layout of its `MYA1` source. It carries the searchable text extracted on-device from an attachment (PDF text-layer / Word `.docx` / text-like), fed into the search index so a note becomes findable by the contents of the files embedded in it.

**Reference implementation:** `protocol/src/attachment_text_blob.rs::{encode_attachment_text_blob_raw, decode_attachment_text_blob}`.
**Spec:** `docs/spec/protocol.md` § 9.15 (`MYE1` outer envelope + opaque inner plaintext + AAD + size cap + cache semantic) + § 3.4 registry entry `"myne.v0.1 attachment-text-key"` (the KEK role) + § 5.2 / § 9.15.3 `AAD_attachment_text_blob` formula (the AAD role `"attachment-text-blob"`, which also binds `attachment_id` + `extractor_version`).

## Opaque inner plaintext

`MYE1` is **payload-agnostic**: the inner plaintext is an opaque byte string the desktop layer produces (the `extractor_version 0x01` format is a postcard-serialized `{status, source_format, text, char_count, truncated}` struct — the NFC-normalized extracted text plus its classification). The protocol crate owns only the crypto framing — magic, version bytes, AEAD, AAD — while the extracted-text serialization is the desktop layer's concern, versioned by the `extractor_version` byte at offset 5. So, like `vault-search` / `vault-dictionary`, there is **no canonical-JSON inner shape** to pin: these vectors treat the plaintext as raw bytes and pin byte-transparency across the AEAD envelope.

## Two distinct load-bearing role strings + two extra AAD bindings

A porter must not conflate the two role strings — they are different constants bound at different layers:

- **KEK (HKDF) role** `"attachment-text-key"` (19 ASCII bytes) — `attachment_text_key = DeriveSubkey(root_key, "attachment-text-key")` per § 3.2 + § 3.4. Pinned independently in the `hkdf-sha256` category. One KEK serves **every** attachment-text blob.
- **AAD role** `"attachment-text-blob"` (20 ASCII bytes, length-prefixed) — the role inside `AAD_attachment_text_blob` per § 5.2 / § 9.15.3. Pinned in the `ctx-aead` / `aad-construction` categories.

Beyond `account_number_hash`, this AAD binds **two** values that the singleton-blob families (prefs/search/dictionary) do not:

- **`attachment_id`** (32 B) — makes a single KEK safe across all attachment-text blobs: a blob sealed for attachment A fails AEAD verification if served as attachment B. The `decrypt-wrong-attachment-id` case pins this.
- **`extractor_version`** (1 B) — bound in the AAD as well as at header offset 5, so a tampered header byte cannot be silently accepted; a future extractor generation (e.g. the OCR layer, MYND-441) bumps it. The `parse-fail-extractor-version-unsupported` case pins the reject.

## Pipeline (as fixed by the reference implementation)

**Encrypt side** (`encode_attachment_text_blob_raw`):

1. **Derive `attachment_text_key`** via the canonical helper: `derive_subkey(root_key, SubkeyRole::AttachmentTextKey)`. HKDF-SHA-256 with `info = "myne.v0.1 attachment-text-key"`, `salt = zero_32`. 32 bytes, deterministic.
2. **Build the AAD** via `aad_attachment_text_blob(account_number_hash, attachment_id, extractor_version)`: 82-byte concatenation of `"myne.v0.1|"` (10 B) + protocol_version `0x01` (1 B) + length-prefixed UTF-8 role `"attachment-text-blob"` (2-B BE length `0x0014` + 20 ASCII bytes = 22 B) + `account_number_hash` (16 B) + `attachment_id` (32 B) + `extractor_version` (1 B) per spec § 9.15.3.
3. **AEAD-seal** the opaque plaintext under `attachment_text_key` + `nonce` + AAD using XChaCha20-Poly1305 with the CTX (Chan & Rogaway 2022) outer commitment per spec § 5.1. Output is `plaintext.len() + 32` bytes.
4. **Assemble the outer envelope** per spec § 9.15.1 fixed-offset layout (total `34 + ciphertext_len` bytes):
   - `[0..4]` magic `MYE1`
   - `[4]` protocol_version `0x01`
   - `[5]` extractor_version `0x01` (the v0.1 baseline extractor generation — a future extractor bumps this so an old decoder rejects cleanly → re-extract from the `MYA1` source)
   - `[6..30]` `nonce` (24 B)
   - `[30..34]` `ciphertext_len` (big-endian uint32, `plaintext.len() + 32`)
   - `[34..]` ciphertext + 32-byte CTX commitment tag

**Decrypt side** (`decode_attachment_text_blob`): the inverse — parse header, validate magic / protocol_version / **extractor_version** (all three checked **before** any decryption per the § 9.15.1 magic-byte-before-decrypt discipline), read `nonce` + `ciphertext_len` (reject `> 4 MiB + 32` cap), rebuild AAD from `account_number_hash` + `attachment_id` + `extractor_version`, AEAD-decrypt. AEAD failure (wrong key / wrong AAD — including a wrong `attachment_id` — / tampered ciphertext) collapses to a single non-distinguishing `CannotDecrypt` per spec § 6.4.

The encode side is **pure and deterministic** given the caller-supplied `nonce`. Production callers use the `encode_attachment_text_blob` convenience wrapper that pulls a nonce from `getrandom`; vectors pin a constant nonce for reproducibility.

## Cache semantic (read side)

`MYE1` is a **rebuildable cache** (like `vault.search`): on any decode failure the desktop caller treats it as a cache miss and re-extracts from the `MYA1` source (§ 9.15.4). The `extractor_version` reject is the forward hook — a `0x01`-only client rejects a future generation's blob and re-extracts.

## Case kinds

| `case_kind` | Pins |
|---|---|
| `encrypt` | the encoder's exact byte output (header + ciphertext + CTX tag) |
| `decrypt_ok` | a successful round-trip back to the opaque plaintext |
| `decrypt_aead_fail` | the non-distinguishing `CannotDecrypt` surface — tag tamper, cross-vault `account_number_hash`, and **wrong `attachment_id`** (all indistinguishable per § 6.4) |
| `parse_fail` | a distinguishable pre-decryption structural reject, one per `AttachmentTextBlobError` decode variant (`magic_mismatch`, `protocol_version_mismatch`, `extractor_version_unsupported`, `header_too_short`, `ciphertext_truncated`, `ciphertext_len_overflow`) |

## File layout

`vectors/attachment-text/v0.1/cases.json` — validated against `schema/v0.1/attachment-text.json` (dispatched from `schema/v0.1.json` by the `schema` field). Regenerate via `cargo test -p protocol --lib emit_attachment_text_vectors_for_paste -- --ignored` (generator in `protocol/src/attachment_text_blob.rs`); the main-repo integration runner is `attachment_text_vectors` in `protocol/tests/vectors.rs` (skips if this fixture is absent).
