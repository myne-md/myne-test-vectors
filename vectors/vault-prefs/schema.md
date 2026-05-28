# `vault-prefs` vector schema

Pins the byte mapping of `protocol::encode_prefs_blob_raw` + `protocol::decode_prefs_blob` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)). The codec emits and consumes `vault.prefs` — the v0.1 vault-level singleton encrypted-preferences blob (MYP1 magic) that lives next to `vault.meta` and carries user-controllable preferences (Phase 1: the screen-capture-protection toggle from MYND-55 / MYND-160).

**Reference implementation:** `protocol/src/prefs.rs::{encode_prefs_blob_raw, decode_prefs_blob}`.
**Spec:** `docs/spec/protocol.md` § 9.5 (`vault.prefs` outer envelope + inner canonical JSON + AAD + size cap + migration-less semantic) + § 3.4 registry entry `"myne.v0.1 prefs-key"` + § 5.2 `AAD_prefs_blob` formula.

## Pipeline (as fixed by the reference implementation)

**Encrypt side** (`encode_prefs_blob_raw`):

1. **Derive `prefs_key`** via the canonical helper: `derive_subkey(root_key, SubkeyRole::PrefsKey)`. HKDF-SHA-256 with `info = "myne.v0.1 prefs-key"`, `salt = zero_32`. Output is 32 bytes, deterministic. The same `root_key` always produces the same `prefs_key`.
2. **Canonical-JSON encode the `PrefsView`.** v0.1's single-field schema renders as `{"screen_capture_protection":true}` or `{"screen_capture_protection":false}` — lexicographically sorted keys, no whitespace, UTF-8 NFC strings, no floating-point per spec § 9.5.2.
3. **Build the AAD** via `aad_prefs_blob(account_number_hash)`: 39-byte concatenation of `"myne.v0.1|"` (10 B) + protocol_version `0x01` (1 B) + length-prefixed UTF-8 role `"prefs-blob"` (2-B BE length + 10 ASCII bytes = 12 B) + `account_number_hash` (16 B) per spec § 9.5.3.
4. **AEAD-seal** the canonical JSON plaintext under `prefs_key` + `prefs_nonce` + AAD using XChaCha20-Poly1305 with the CTX (Chan & Rogaway 2022) outer commitment per spec § 5.1. Output is `plaintext.len() + 32` bytes (CTX tag is 32 bytes).
5. **Assemble the outer envelope** per spec § 9.5.1 fixed-offset layout (total `34 + ciphertext_len` bytes):
   - `[0..4]` magic `MYP1`
   - `[4]` protocol_version `0x01`
   - `[5]` reserved `0x00`
   - `[6..30]` `prefs_nonce` (24 B)
   - `[30..34]` `ciphertext_len` (big-endian uint32, `plaintext.len() + 32`)
   - `[34..]` ciphertext + 32-byte CTX commitment tag

**Decrypt side** (`decode_prefs_blob`): the inverse — parse header, validate magic / version / reserved, read `prefs_nonce` + `ciphertext_len` (reject `> 65536 + 32` cap per spec § 9.5.4), rebuild AAD from `account_number_hash`, AEAD-decrypt, canonical-JSON parse. AEAD failure (wrong key / wrong AAD / tampered ciphertext) collapses to a single non-distinguishing `CannotDecrypt` per spec § 6.4 + § 9.5.5.

The encode side is **pure and deterministic** given the caller-supplied `prefs_nonce` (no internal CSPRNG, no clock) — same inputs produce identical output bytes. Production callers use the `encode_prefs_blob` convenience wrapper that pulls a nonce from `getrandom`; vectors pin a constant nonce for reproducibility.

## File layout

```
vectors/vault-prefs/
├── schema.md              ← this file
└── v0.1/
    └── cases.json         ← array of 5 KAT cases (encrypt + decrypt + failure-mode)
```

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/vault-prefs.json`](../../schema/v0.1/vault-prefs.json) (per-case shape with a 3-way `case_kind` discriminator: `encrypt`, `decrypt_ok`, `decrypt_aead_fail`). Run the repository's documented `ajv-cli validate` command (see the root `README.md`) to verify any `cases.json` against the canonical schema. The sub-schema is authoritative — this document covers only the domain-specific semantics.

Three case kinds carry different field shapes:

- **`encrypt`** — pins the encoder's byte output. Inputs: `input_root_key_hex` (32 B), `input_account_number_hash_hex` (16 B), `input_plaintext_canonical_utf8_hex`, `input_prefs_nonce_hex` (24 B). Optional `informational_prefs_key_hex` carries the pre-computed HKDF output for reviewer convenience. Output: `expected_prefs_blob_hex` (the 100-byte `MYP1` blob = 34-header + 66-ciphertext+tag).
- **`decrypt_ok`** — pins the decoder's successful round-trip. Inputs: `input_blob_hex` + correct `input_root_key_hex` + correct `input_account_number_hash_hex`. Outputs: `expected_plaintext_canonical_utf8_hex` (raw bytes) and `expected_prefs_view` (the parsed `PrefsView` object — implementations parse the recovered JSON and assert structural equality).
- **`decrypt_aead_fail`** — pins the non-distinguishing failure surface. Inputs: `input_blob_hex` + (intentionally wrong or tampered) `input_root_key_hex` + `input_account_number_hash_hex`. Output: `expected_kind: "aead_fail"`. The decoder MUST collapse cross-vault, wrong-AAD, ciphertext-tamper, and CTX-tamper to a single indistinguishable failure (spec § 6.4 + § 9.5.5).

## How to consume

A port runs each case based on `case_kind`:

- **`encrypt`**: decode every `input_*_hex`, build a `PrefsView` from the implied plaintext shape, invoke the port's `encode_prefs_blob_raw` equivalent with the supplied nonce, hex-encode the output, compare byte-equal to `expected_prefs_blob_hex`. Any non-equal result is a stop-the-world bug (per repo `README.md` "Cross-implementation discipline").
- **`decrypt_ok`**: decode `input_blob_hex` + key + account hash, invoke the port's `decode_prefs_blob` equivalent, hex-encode the recovered plaintext, compare to `expected_plaintext_canonical_utf8_hex`, AND parse the canonical JSON and compare to `expected_prefs_view`.
- **`decrypt_aead_fail`**: decode inputs, invoke decoder, assert it returns a non-distinguishing AEAD failure (whatever the port's equivalent is — `Err(AeadFailure)`, exception, error code). The port MUST NOT distinguish which failure arm fired beyond "this blob cannot be decrypted with these inputs".

## Why hex, not JSON-native UTF-8 strings or numeric arrays

JSON natively supports UTF-8 strings via `\uXXXX` escapes, but escape behavior varies subtly across language libraries. JSON also supports numeric arrays for byte sequences, but byte-array parsing varies and is awkward to diff. Hex-encoding the raw bytes eliminates parser-side transformations: bytes that go in are exactly the bytes the reference implementation saw.

## Invariants the vectors pin

For the 5 cases in `cases.json`:

- **MYP1 magic + version + reserved bytes.** Encoder output `[0..4]` = `MYP1`, `[4]` = `0x01`, `[5]` = `0x00`. Decoder rejects any other value at parse time before reaching AEAD verification.
- **Header byte layout.** Encoder output `[6..30]` = supplied `prefs_nonce` (24 B); `[30..34]` = `ciphertext_len` as big-endian uint32; `[34..]` = ciphertext + 32-byte CTX tag.
- **Canonical JSON discipline.** v0.1 plaintext is exactly `{"screen_capture_protection":true}` (34 bytes) or `{"screen_capture_protection":false}` (35 bytes). No whitespace, no key reordering, no trailing characters, no leading BOM.
- **AAD binds account identity, not items_key lineage.** Cross-vault attack (case 4): a blob encoded under vault A's `account_number_hash` MUST fail to decrypt under vault B's `account_number_hash`, even if `prefs_key` somehow matched. The AAD difference is structural defense alongside the key mismatch.
- **Tamper rejection (case 5).** Flipping a single bit anywhere in the ciphertext / CTX tag MUST cause the decoder to return a non-distinguishing AEAD failure — the CTX commitment construction guarantees this without depending on the underlying Poly1305 tag's malleability properties.
- **Non-distinguishing failure surface (cases 4 + 5).** A port's decoder MUST emit identical observable behavior for cross-vault AND tampered cases — both are "AEAD failure"; implementations MUST NOT leak which failure mode triggered.

## Regeneration

If the reference implementation changes (must not happen byte-stably without a major-version event), regenerate this vector via:

```bash
cargo test -p protocol --lib emit_prefs_vectors_for_paste -- --ignored
cat /tmp/myne_vault_prefs_vectors.txt
# paste hex outputs into cases.json
```

The regenerator test lives in `protocol/src/prefs.rs::tests::emit_prefs_vectors_for_paste`. It is `#[ignore]`-gated so it never runs in default `cargo test` invocations.

## What this does NOT cover

- The full `vault.meta` serialization vectors (`vectors/vault_meta_rewrite/`) — those pin the password-rewrite path; this directory pins only the standalone vault.prefs codec.
- The Tauri-layer `prefs_read` / `set_screen_capture_protection` IPC commands — those wrap this codec with Slot::Unlocked-gating, account-hash extraction from vault.meta, and atomic disk write. Wrapper-layer behavior is verified by `src-tauri/src/plugins/prefs/commands.rs` integration tests, NOT cross-implementation vectors (the wrapper is Rust/Tauri-specific).
- The native `screen_capture::apply` side-effect — platform-specific, no cross-implementation contract.
- Schema evolution within v0.1 (additive optional fields per spec § 9.5.2 forward-compat). When v0.1.x ships a second pref field, this vector set extends with a new case rather than mutating existing entries (immutable `vector_id`s).
