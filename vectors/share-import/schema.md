# `share-import` vector schema

Pins the byte mapping of `protocol::encode_share_import_blob_raw` + `protocol::decode_share_import_blob` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)). The codec emits and consumes `pending-shares/<share-id>` — the v0.1 **device-local share-import staging** blob (`MYF1` magic; OS share-sheet file import). A file handed to Myne from the OS share sheet (iOS Share Extension / Android `ACTION_SEND`) is held in a device-local encrypted queue while the process lacks `root_key` (the iOS extension is a separate sandbox; on Android the vault may be locked), then drained into the vault's attachment library (§ 9.9 / § 9.10) via `attachment_import` on the next unlock.

**Reference implementation:** `protocol/src/share_import_blob.rs::{encode_share_import_blob_raw, decode_share_import_blob, parse_share_import_header}`.
**Spec:** `docs/spec/protocol.md` § 9.19 (`pending-shares/<id>` layout + variable-length ciphertext + AAD + device-key model) + § 5.2 / § 9.19.3 `AAD_share_import_blob` formula (the AAD role `"share-import-blob"`).

## `MYF1` magic allocation

The 4-byte magic `MYF1` (`4d 59 46 31`) is a free-pool § 10.3 allocation (`F` = file, MYND-507), implementer-chosen the same way every family's magic is (cf. `MYK1`'s "K because C is taken"). The vectors are generated with `MYF1`; a later letter change would regenerate this category's `expected_*_blob_hex` bytes and the layout below in lockstep (a `vector_id` is never reused, so a magic change is a fresh case set, not an edit).

## The file-carrying sibling of `pending-capture`

`share-import` is the structural twin of the `pending-capture` (`MYK1`, § 9.18) family — same 49-byte header, same device-local `wrap_key` seal, same vault-agnostic per-entry-id AAD binding, same variable ciphertext with a cap-before-allocation check. The two differences:

- the plaintext is **arbitrary binary** (a shared file's bytes, framed with its filename + declared MIME by the core layer, `myne_core::plugins::share_import`) rather than dictated UTF-8 text — so this codec is byte-transparent over genuinely non-UTF-8 input;
- the cap is **64 MiB** (`MAX_SHARE_IMPORT_PLAINTEXT_LEN`, matching the `MYA1` attachment cap) rather than 16 KiB.

## No HKDF subkey role, and no vault identity — the vault-agnostic blob

Every § 9 **vault** blob seals under `DeriveSubkey(root_key, …)` and binds a vault identity (`account_number_hash` or `items_key_id`). `pending-shares/<id>` does **neither**: it is created while the vault is **locked** (or from a separate share-extension sandbox that does not hold `root_key`, so the target vault is unknown), sealed under a **device-local `wrap_key`** held in the OS keychain / Android Keystore. So:

- there is **no `hkdf-sha256` role** for share-import (do not look for one);
- the vectors supply `wrap_key` **directly** as a fixed fixture;
- no `account_number_hash` / `items_key_id` is bound — the only bound identity is the per-entry `share_id`.

The only load-bearing role string is the **AAD role** `"share-import-blob"` (17 ASCII bytes, length-prefixed). It is pinned by the `aad_share_import_blob_default_matches_pinned_hex` unit test in the main repo plus the `encrypt` cases here; the device-local, vault-agnostic AAD is deliberately **not** enrolled in the vault-scoped `ctx-aead` / `aad-construction` builder-enumeration families.

## `share_id` is AAD-bound (swap/duplicate protection)

`share_id` (16 random bytes) lives in the public header at offset 5 **and** is bound into the 49-byte AAD via `aad_share_import_blob(share_id)`. Because a single device key protects every queue entry, this binding is what stops an attacker with write access to the queue directory from swapping or duplicating one entry's ciphertext into another `share_id` slot: flipping a header `share_id` byte passes the structural header checks but fails AEAD verification (a `decrypt_aead_fail`, not a `parse_fail`).

## Variable ciphertext length with a 64 MiB cap

Unlike `vault-quickunlock`'s fixed 64-byte ciphertext, a shared file is arbitrary-length, so `ciphertext_len` (= `plaintext_len + 32`) is a real field. The decoder rejects `ciphertext_len > 67108864 + 32` **before** allocating — a `ciphertext_len_overflow` `parse_fail` (contrast `vault-quickunlock`'s `ciphertext_len_invalid`, which rejects any value != the fixed 64).

## Layout (as fixed by the reference implementation)

```text
Offset  Size  Field             Notes
0       4     magic             ASCII "MYF1"
4       1     protocol_version  0x01
5       16    share_id          16 random bytes; ALSO the queue filename; AAD-bound
21      24    wrap_nonce        XChaCha20-Poly1305 nonce
45      4     ciphertext_len    big-endian uint32, = plaintext_len + 32
49      ..    ciphertext        framed file record AEAD-sealed + 32-byte CTX tag
```

## Case kinds

- **`encrypt`** — pin the encoder's byte output for a given `wrap_key` + `share_id` + `plaintext` + `nonce`. 2 cases (short binary + genuinely non-UTF-8 binary, to pin byte transparency + a different variable `ciphertext_len`).
- **`decrypt_ok`** — round-trip a blob back to its framed-file plaintext under the matching `wrap_key` (the `share_id` is read from the blob's own header). 1 case.
- **`decrypt_aead_fail`** — the non-distinguishing `CannotDecrypt` surface (§6.4): CTX-tag tamper, cross-`share_id` (AAD-bound header flip), wrong `wrap_key`. 3 cases.
- **`parse_fail`** — a distinguishable pre-decryption structural reject, one per `ShareImportBlobError` decode variant: `magic_mismatch`, `protocol_version_mismatch`, `ciphertext_len_overflow`, `header_too_short`, `ciphertext_truncated`. 5 cases.

11 cases total. Generator: `emit_share_import_blob_vectors_for_paste` (`protocol/src/share_import_blob.rs`); runner: `share_import_vectors` (`protocol/tests/vectors.rs`).
