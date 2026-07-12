# `pending-capture` vector schema

Pins the byte mapping of `protocol::encode_capture_blob_raw` + `protocol::decode_capture_blob` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)). The codec emits and consumes `captures/<capture-id>` — the v0.1 **device-local pending-capture queue** blob (`MYK1` magic; Apple Watch voice quick-capture). A watch-dictated note is relayed to the phone over WatchConnectivity and held in a device-local encrypted queue while the vault is locked, then drained into the vault as an ordinary note on unlock.

**Reference implementation:** `protocol/src/capture_blob.rs::{encode_capture_blob_raw, decode_capture_blob, parse_capture_header}`.
**Spec:** `docs/spec/protocol.md` § 9.18 (`captures/<id>` layout + variable-length ciphertext + AAD + device-key model) + § 5.2 / § 9.18.3 `AAD_pending_capture_blob` formula (the AAD role `"pending-capture-blob"`).

## No HKDF subkey role, and no vault identity — the vault-agnostic blob

Every § 9 **vault** blob seals under `DeriveSubkey(root_key, …)` and binds a vault identity (`account_number_hash` or `items_key_id`). `captures/<id>` does **neither**: it is created while the vault is **locked** (so `root_key` is unavailable and the target vault is unknown), sealed under a **device-local `wrap_key`** held in the OS keychain (service `md.myne.captures`, non-biometric, `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`). So:

- there is **no `hkdf-sha256` role** for pending-capture (do not look for one);
- the vectors supply `wrap_key` **directly** as a fixed fixture;
- no `account_number_hash` / `items_key_id` is bound — the only bound identity is the per-entry `capture_id`.

The only load-bearing role string is the **AAD role** `"pending-capture-blob"` (20 ASCII bytes, length-prefixed). It is pinned by the `aad_capture_blob_default_matches_pinned_hex` unit test in the main repo plus the `encrypt` cases here; the device-local, vault-agnostic AAD is deliberately **not** enrolled in the vault-scoped `ctx-aead` / `aad-construction` builder-enumeration families.

## `capture_id` is AAD-bound (swap/duplicate protection)

`capture_id` (16-byte UUIDv7) lives in the public header at offset 5 **and** is bound into the 49-byte AAD via `aad_capture_blob(capture_id)`. Because a single device key protects every queue entry, this binding is what stops an attacker with write access to the queue directory from swapping or duplicating one entry's ciphertext into another `capture_id` slot: flipping a header `capture_id` byte passes the structural header checks but fails AEAD verification (a `decrypt_aead_fail`, not a `parse_fail`).

## Variable ciphertext length with a 16 KiB cap

Unlike `vault-quickunlock`'s fixed 64-byte ciphertext, a dictated note is arbitrary-length, so `ciphertext_len` (= `plaintext_len + 32`) is a real field. The decoder rejects `ciphertext_len > 16384 + 32` **before** allocating — a `ciphertext_len_overflow` `parse_fail` (contrast `vault-quickunlock`'s `ciphertext_len_invalid`, which rejects any value != the fixed 64).

## Layout (as fixed by the reference implementation)

```text
Offset  Size  Field             Notes
0       4     magic             ASCII "MYK1"
4       1     protocol_version  0x01
5       16    capture_id        random UUIDv7; ALSO the queue filename; AAD-bound
21      24    wrap_nonce        XChaCha20-Poly1305 nonce
45      4     ciphertext_len    big-endian uint32, = plaintext_len + 32
49      ..    ciphertext        dictated UTF-8 AEAD-sealed + 32-byte CTX tag
```

## Case kinds

- **`encrypt`** — pin the encoder's byte output for a given `wrap_key` + `capture_id` + `plaintext` + `nonce`. 2 cases (short ASCII + multibyte UTF-8, to pin byte transparency + a different variable `ciphertext_len`).
- **`decrypt_ok`** — round-trip a blob back to its dictated plaintext under the matching `wrap_key` (the `capture_id` is read from the blob's own header). 1 case.
- **`decrypt_aead_fail`** — the non-distinguishing `CannotDecrypt` surface (§6.4): CTX-tag tamper, cross-`capture_id` (AAD-bound header flip), wrong `wrap_key`. 3 cases.
- **`parse_fail`** — a distinguishable pre-decryption structural reject, one per `CaptureBlobError` decode variant: `magic_mismatch`, `protocol_version_mismatch`, `ciphertext_len_overflow`, `header_too_short`, `ciphertext_truncated`. 5 cases.

11 cases total. Generator: `emit_capture_blob_vectors_for_paste` (`protocol/src/capture_blob.rs`); runner: `pending_capture_vectors` (`protocol/tests/vectors.rs`).
