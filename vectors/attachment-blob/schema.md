# `attachment-blob` vector schema

Pins the byte mapping of `protocol::encode_attachment_blob_raw` + `protocol::decode_attachment_blob` + `protocol::compute_attachment_id` and the two attachment AAD builders (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)). The codec emits and consumes the v0.1 per-attachment encrypted blob (`MYA1` magic, MYND-164) stored at `attachments/<attachment-id-hex>` inside the vault directory.

**Reference implementation:** `protocol/src/attachment_blob.rs::{encode_attachment_blob_raw, decode_attachment_blob, compute_attachment_id, aad_wrap_attachment_key, aad_encrypt_attachment}`.
**Spec:** `docs/spec/protocol.md` § 9.9 (`MYA1` layout + key lineage + 64-MiB cap) + § 5.2 (AAD formulas) + § 3.4 registry entry `"attachments-id-hmac"` (the id-HMAC subkey role).

## Key lineage (unlike the vault singletons)

The attachment codec hangs off the **itemsKey lineage**, not a root-key subkey: a fresh random per-attachment `attachment_key` encrypts the file bytes, and that key is wrapped under the vault's `itemsKey`. This mirrors the per-note `.myn` design (`myn-blob` category) rather than the `vault-prefs`/`vault-search`/`attachments-catalog` singletons whose KEKs come from `DeriveSubkey(root_key, …)`:

```
itemsKey ──wrap──▶ attachment_key ──encrypt──▶ file bytes
```

Two AEAD layers, two AADs, both binding `items_key_id` + `attachment_id`; the content layer additionally binds `content_type` (length-prefixed). The 32-byte `attachment_id` is the on-disk filename and is bound into both AADs but **never stored inside the blob bytes** — a swapped/renamed attachment file fails AEAD verification.

## Content-addressed `attachment_id` (the second subkey role)

`compute_attachment_id(root_key, file_plaintext) = HMAC-SHA-256(DeriveSubkey(root_key, "attachments-id-hmac"), file_plaintext)` — deterministic per (vault, content) so re-importing identical bytes dedups to the same on-disk file, while a filesystem adversary cannot link identical files across vaults (the HMAC key is vault-secret). A porter must not conflate the two attachment subkey roles (the § 9.10.1 domain-separation pair, pinned in `hkdf-sha256`):

- `"attachments-id-hmac"` (19 ASCII bytes) — keys this HMAC. **Not** a blob KEK.
- `"attachments-catalog-key"` (23 ASCII bytes) — KEK for the `vault.attachments` catalog (the `attachments-catalog` category).

## Pipeline (as fixed by the reference implementation)

**Encrypt side** (`encode_attachment_blob_raw`):

1. **Wrap `attachment_key`** under `itemsKey` with `aad_wrap_attachment_key(items_key_id, attachment_id)`: `"myne.v0.1|"` (10 B) + protocol_version `0x01` (1 B) + `items_key_id` (16 B) + `attachment_id` (32 B) = 59 bytes. Output `enc_attachment_key` is `32 + 32 = 64` bytes (key + CTX tag).
2. **Encrypt the file bytes** under `attachment_key` with `aad_encrypt_attachment(items_key_id, attachment_id, content_type)`: the same 59-byte prefix + 2-B BE length-prefixed `content_type` bytes. Rejects plaintexts above the 64-MiB cap (`MAX_ATTACHMENT_LEN`).
3. **Assemble the outer layout** per spec § 9.9 fixed offsets (total `140 + ciphertext_len` bytes):
   - `[0..4]` magic `MYA1`
   - `[4]` protocol_version `0x01`
   - `[5]` reserved `0x00`
   - `[6..22]` `items_key_id` (16 B)
   - `[22..46]` `attachment_key_nonce` (24 B)
   - `[46..48]` `enc_attachment_key_len` (big-endian uint16, always `0x0040` = 64)
   - `[48..112]` `enc_attachment_key` (64 B)
   - `[112..136]` `content_nonce` (24 B)
   - `[136..140]` `ciphertext_len` (big-endian uint32, `file_plaintext.len() + 32`)
   - `[140..]` ciphertext + 32-byte CTX commitment tag

**Decrypt side** (`decode_attachment_blob`): parse + validate the header (magic / version / reserved before any decryption), unwrap `attachment_key` under `itemsKey`, decrypt the content layer. Any AEAD failure (wrong key, wrong `attachment_id`, wrong `content_type`, tamper) collapses to the single non-distinguishing `CannotDecrypt` per spec § 6.4.

Both directions are pure and deterministic given the caller-supplied nonces (`_raw` convention); production callers use the `encode_attachment_blob` wrapper that pulls both nonces from `getrandom`.

## File layout

```
vectors/attachment-blob/
├── schema.md              ← this file
└── v0.1/
    └── cases.json         ← array of 9 KAT cases (encrypt + decrypt + AEAD-fail + AAD + id-determinism)
```

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/attachment-blob.json`](../../schema/v0.1/attachment-blob.json) (per-case shape with a 6-way `case_kind` discriminator). The sub-schema is authoritative — this document covers only the domain-specific semantics.

- **`encrypt`** (`encrypt-small`, `encrypt-1kb`) — pins the encoder's full `MYA1` byte output. Inputs: `input_items_key_hex` + `input_items_key_id_hex` + `input_attachment_id_hex` + `input_content_type_utf8_hex` + `input_attachment_key_hex` + both nonces + `input_file_plaintext_hex`. Output: `expected_attachment_blob_hex`.
- **`decrypt_ok`** (`decrypt-ok`) — round-trip: blob + correct itemsKey/ids/content_type → `expected_file_plaintext_hex`.
- **`decrypt_aead_fail`** (`decrypt-fail-tamper`, `decrypt-fail-cross-vault`, `decrypt-items-key-id-mismatch`) — non-distinguishing failure surface (`expected_kind: "aead_fail"`): ciphertext tamper, wrong-vault inputs, and a mismatched caller `items_key_id` against an intact blob MUST all be indistinguishable.
- **`aad_construction`** (`aad-wrap-attachment-key`, `aad-encrypt-attachment`) — the two builders' canonical bytes (`input_aad_builder` names which); `aad_encrypt_attachment` additionally binds the length-prefixed `content_type`.
- **`id_determinism`** (`id-determinism-a`, `id-determinism-b`) — `input_root_key_hex` + `input_file_plaintext_hex` → `expected_attachment_id_hex`; optional `informational_id_hmac_key_hex` carries the derived HMAC key for reviewer convenience.
- **`parse_fail`** (`parse-fail-*`, MYND-227) — the structural reject checks (§ 9.9 layout + strict-trailing): header-too-short, magic, protocol_version, reserved byte, `enc_attachment_key_len` ≠ 64, `ciphertext_len` below-tag / overflow, trailing byte. `expected_kind: "parse_fail"` — the SAME non-distinguishing `CannotDecrypt` as the AEAD surface (the case_kind documents WHICH checks a port must perform, not a distinguishable error). The cases carry the REAL `items_key` deliberately: header bytes are not AAD-bound, so a port that skips a structural check decodes the tampered blob successfully and returns Ok — which the runner catches.

## How to consume

A port runs each case based on `case_kind`: invoke the equivalent encoder/decoder/AAD-builder/id-computation with the supplied inputs and compare byte-equal (or assert the non-distinguishing AEAD failure). Any mismatch is a stop-the-world bug (per repo `README.md` "Cross-implementation discipline").

## Invariants the vectors pin

- **MYA1 magic + version + reserved bytes** and the full fixed-offset header layout above, including the always-64 `enc_attachment_key_len` field.
- **Two-layer key wrap byte mapping.** `enc_attachment_key` and the content ciphertext are pinned in one output; a port that wraps with the wrong AAD or layers in the wrong order produces different bytes.
- **`attachment_id` binding without storage.** The id appears in both AADs but not in the blob; `decrypt-fail-cross-vault` pins that wrong identity inputs fail AEAD.
- **Tamper rejection + non-distinguishing surface.** The `decrypt_aead_fail` cases MUST be indistinguishable from each other (spec § 6.4) — including the `items_key_id`-mismatch case, which pins that the embedded-vs-caller id comparison happens at all (skipping it would let an intact blob decode under the right key).
- **Content-addressed id determinism.** Same (root_key, bytes) → same id; different bytes → different id; the HMAC key derives via the canonical `DeriveSubkey` helper only.
- **Structural reject checks (MYND-227, `parse-fail-*`).** Every § 9.9 layout check is pinned by a tamper that the AEAD tags alone would NOT reject (the fixtures stay otherwise valid) — sharpest on `parse-fail-trailing-byte`, where strict-trailing length discipline is the only thing standing between a 1-byte-appended blob and a successful decode. The `ciphertext-len-overflow` case pins the 64 MiB + 32 cap from the read direction with a normal-sized fixture.

## Regeneration

```bash
cargo test -p protocol --lib emit_attachment_blob_vectors_for_paste -- --ignored
cat /tmp/myne_attachment_blob_vectors.json
# copy the full JSON verbatim into cases.json
```

The regenerator lives in `protocol/src/attachment_blob.rs`; it is `#[ignore]`-gated so it never runs in default `cargo test` invocations.

The MYND-227 `parse_fail` family (8 cases + the `decrypt-items-key-id-mismatch` aead-fail case) has its own append generator (the original stays byte-stable — vector_id immutability):

```bash
cargo test -p protocol --lib emit_attachment_blob_parse_fail_vectors -- --ignored
# splice the JSON array fragment at /tmp/myne_attachment_blob_parse_fail_vectors.json into the cases array
```

## What this does NOT cover

- **Encode-only error modes.** `AttachmentTooLarge` (above the 64-MiB cap) and `RandomnessUnavailable` (CSPRNG failure in the non-`_raw` wrapper) cannot be expressed as decode-a-blob cases — a real ≥64 MiB fixture would balloon cases.json by ~128 MB of hex. The cap IS pinned from the read direction (MYND-227): `parse-fail-ciphertext-len-overflow` advertises `0xffffffff` in a normal-sized blob, so a port MUST bound-check before allocating.
- **The `vault.attachments` catalog** — its codec + canonical JSON live in the sibling `attachments-catalog` category.
- **The Tauri-layer `attachment://` serve scheme, editor embeds, media ingest** (MYND-164 desktop side) — verified by `src-tauri` integration tests, not cross-implementation vectors.
- **Filename/path discipline** (`attachments/<attachment-id-hex>`) — a host-layer storage concern (spec § 9.1), not a byte-level codec surface.
