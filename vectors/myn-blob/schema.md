# `myn-blob` vector schema

Pins the byte mapping of `protocol::encode_new_note_blob_raw` + `protocol::decode_note` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)) — the `MY01` per-note blob encoder/decoder pair. Each blob holds an AEAD-wrapped `item_key` (under the singleton `itemsKey`) plus AEAD-encrypted content (under `item_key`), with two layers of CTX-committed XChaCha20-Poly1305 sealing.

**Reference implementation:** `protocol/src/blob.rs::{encode_new_note_blob_raw, decode_note}`.
**Spec:** `docs/spec/protocol.md` § 9.2 (on-disk byte layout) + § 6.4 (decode failure surface) + § 6.5 (AAD discipline) + § 5.2 (AAD construction).

## Pipeline (as fixed by spec § 9.2)

**Encrypt side** (`encode_new_note_blob_raw`):

1. **Wrap `item_key` under `itemsKey`** via `aead_seal_ctx` with AAD constructed by `aad_note_item_key_wrap(items_key_id, note_uuid)`. Output: 64-byte ciphertext (32-byte wrapped `item_key` + 32-byte CTX tag).
2. **Encrypt plaintext under `item_key`** via `aead_seal_ctx` with AAD constructed by `aad_note_content_encrypt(items_key_id, note_uuid)`. **`content_type` is hardcoded** to `b"text/markdown"` inside the AAD builder — v0.1 has no per-vector parameter. Output: `plaintext.len() + 32` byte ciphertext (plaintext + CTX tag).
3. **Assemble the `MY01` byte sequence** per spec § 9.2 fixed-offset layout. Total: `76 + plaintext.len() + 32` bytes.

**Decrypt side** (`decode_note`): the inverse — parse header (magic + version + reserved + items_key_id + item_key_nonce + ciphertext_len), check magic/version/reserved/length bounds, AEAD-unwrap `item_key` under `itemsKey` (step 1 reverse), AEAD-decrypt content under `item_key` (step 2 reverse). Failure at ANY stage collapses to the single non-distinguishing `DecryptError::CannotDecrypt` per spec § 6.4.

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/myn-blob.json`](../../schema/v0.1/myn-blob.json) (per-case shape with a 3-way `case_kind` discriminator: `encrypt`, `decrypt_ok`, `decrypt_aead_fail`). Run the repository's documented `ajv-cli validate` command (see the root `README.md`) to verify any `cases.json` against the canonical schema. The sub-schema is authoritative — this document covers only the domain-specific semantics.

Three case kinds carry different field shapes:

- **`encrypt`** — pins the encoder's byte output. Inputs: `items_key`, `items_key_id`, `note_uuid`, `item_key`, `item_key_nonce`, `content_nonce` (each hex), plus `input_plaintext_hex` (variable length). Output: `expected_myn_blob_hex` (full `MY01` blob bytes).
- **`decrypt_ok`** — pins the decoder's successful round-trip. Inputs: `input_blob_hex` + the correct `note_uuid`/`items_key`/`items_key_id`. Output: `expected_plaintext_hex` (the recovered plaintext bytes).
- **`decrypt_aead_fail`** — pins the non-distinguishing AEAD-stage failure surface. Inputs: a tampered or wrong-key blob + the supplied note context. Output: `expected_kind: "aead_fail"`. Implementations MUST collapse all rejection causes (wrong key / wrong AAD / tampered ciphertext / tampered tag) to their port's equivalent of `Err(DecryptError::CannotDecrypt)` per spec § 6.4.

## How to consume

A port runs each case based on `case_kind`:

- **`encrypt`**: decode every `input_*_hex`, invoke the port's `encode_new_note_blob_raw` equivalent with the supplied inputs, hex-encode the output, compare byte-equal to `expected_myn_blob_hex`. Any divergence is a stop-the-world bug.
- **`decrypt_ok`**: decode `input_blob_hex` + the note context, invoke the port's `decode_note` equivalent, hex-encode the recovered plaintext, compare byte-equal to `expected_plaintext_hex`. Implementations SHOULD also verify the recovered `items_key_id` + `note_uuid` match the inputs (catches a class of AAD-binding bugs).
- **`decrypt_aead_fail`**: decode inputs, invoke decoder, assert the port returns its non-distinguishing aead-fail variant. Ports MUST NOT distinguish which arm fired beyond a single 'cannot decrypt' outcome at the caller-visible surface.

## Invariants the vectors pin

For the 5 cases in `cases.json`:

- **`MY01` magic + version + reserved bytes.** Encoder output `[0..4]` = `MY01`, `[4]` = `0x01`, `[5]` = `0x00`. Decoder rejects any other value at parse time before reaching AEAD verification.
- **Header byte layout (76-byte prefix).** Encoder output `[6..22]` = `items_key_id`, `[22..46]` = `item_key_nonce`, `[46..48]` = wrapped `item_key` length BE-u16 (always 0x0040 = 64 in v0.1), `[48..112]` = wrapped `enc_item_key`, `[112..136]` = `content_nonce`, `[136..140]` = `ciphertext_len` BE-u32.
- **Plaintext size coverage.** The 3 encrypt cases pin the encoder at 10 B, 1 KB, and 100 KB plaintext sizes — proves the encoder remains byte-correct across realistic-note ranges without hitting the `MAX_PLAINTEXT_LEN` (16 MiB) cap.
- **Round-trip property.** The `decrypt-ok-1kb` case proves the decoder recovers the canonical plaintext from the 1-KB encrypt-case blob byte-for-byte.
- **Tamper rejection (`decrypt-tampered-ciphertext`).** Flipping a single byte at `OFFSET_CIPHERTEXT` (first byte of encrypted content) in the 10-byte encrypt-case blob MUST cause the decoder to return non-distinguishing `DecryptError::CannotDecrypt`. The inner CTX commitment construction over the content layer catches the modification — the recomputed Poly1305 tag changes, the recomputed CTX commitment changes, the constant-time comparison fails.
- **content_type hardcoded.** v0.1 binds `content_type = "text/markdown"` inside `aad_note_content_encrypt`. There's no per-vector parameter; cross-impl ports MUST mirror this constant.

## Regeneration

If the reference implementation changes (must not happen byte-stably without a major-version event), regenerate this vector set via:

```bash
cargo test -p protocol --lib emit_myn_blob_vectors_for_paste -- --ignored
cp /tmp/myne_myn_blob_vectors.json vectors/myn-blob/v0.1/cases.json
```

The regenerator test lives in `protocol/src/blob.rs::tests::emit_myn_blob_vectors_for_paste`. `#[ignore]`-gated. Emits ready-to-copy JSON. Output file is ~425 KB due to the 100-KB plaintext case inlined as a ~205-KB hex string — git-friendly, `jq`-readable.

## What this does NOT cover

- **`encode_update_blob_raw` / `encode_update` / `encode_new_note`** (the higher-level wrappers that pull randomness internally). The pure deterministic `_raw` path is the cross-implementation contract; the random-source wrappers are Rust/Tauri-specific ergonomics.
- **AAD construction byte details.** The actual `aad_note_item_key_wrap` + `aad_note_content_encrypt` byte assembly is covered by the future `vectors/aad-construction/` category (not yet a Plane task — placeholder in companion CLAUDE.md). This category's vectors exercise the AAD bytes implicitly (any AAD drift surfaces as a `decode_note` rejection in the round-trip case).
- **`items_key`-side mismatches.** Cross-vault scenarios (where `items_key` mismatches between encode and decode) are structurally equivalent to the tampering case — both collapse to `CannotDecrypt`. A dedicated cross-vault case could be added in a future expansion if observability needs warrant.
- **Content-type variations.** v0.1 has no per-vector content_type parameter; the value is hardcoded `text/markdown`. A future spec revision that introduces content-type negotiation would land vectors here.
- **`MAX_PLAINTEXT_LEN` cap probe.** The 16 MiB cap is exercised by in-crate unit tests; cross-impl conformance at the cap is high-value but at ~32 MB hex inline cases.json would balloon. Deferred to a future expansion if cross-impl drift is observed at the cap boundary.
