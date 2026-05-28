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

Top-level object:

| Field | Type | Description |
|---|---|---|
| `schema` | string | Always `"vault-prefs"` for this directory. |
| `protocol_version` | string | `"v0.1"` for v0.1 vectors. |
| `spec_section` | string | Anchors the canonical pipeline definition (`"protocol.md §9.5"`). |
| `description` | string | Human-readable summary of what is pinned. |
| `cases` | array | One element per test case (see below). 5 cases total per MYND-160 grill Q6.2. |

Each `cases[i]` element carries a `case_kind` discriminator and a subset of fields. Three kinds:

### `case_kind = "encrypt"` — pin the encoder's byte output

| Field | Type | Description |
|---|---|---|
| `vector_id` | string | Stable identifier (`"vault-prefs/v0.1/<slug>"`). Immutable once shipped. |
| `description` | string | What the case verifies. |
| `case_kind` | string | `"encrypt"`. |
| `input_root_key_hex` | string | 32-byte `root_key` (hex). The encoder derives `prefs_key` from this. |
| `input_account_number_hash_hex` | string | 16-byte `account_number_hash` (hex) — the same field stored at `vault.meta` offset 6, used to build the AAD. |
| `input_plaintext_canonical_utf8_hex` | string | The exact canonical-JSON plaintext bytes (hex). Implementations MUST emit the same UTF-8 bytes from `PrefsView::default()` (or the toggled-off equivalent) — drift here means the canonical encoder is wrong. |
| `input_prefs_nonce_hex` | string | 24-byte XChaCha20-Poly1305 nonce (hex). Production callers source from OS CSPRNG; vectors pin a constant. |
| `informational_prefs_key_hex` | string | The 32-byte HKDF output (hex), pre-computed. Informational only — the encoder derives this from `input_root_key_hex` + the registered context string; vectors carry it so reviewers don't have to re-derive to verify intermediate state. |
| `expected_blob_hex` | string | Lowercase hex of the full 100-byte (34-header + 66-ciphertext+tag) blob. Implementations decode every `*_hex` input field, run their encoder, hex-encode the output, compare byte-equal. |

### `case_kind = "decrypt_ok"` — pin the decoder's successful round-trip

| Field | Type | Description |
|---|---|---|
| `vector_id` | string | Stable identifier. |
| `description` | string | What the case verifies. |
| `case_kind` | string | `"decrypt_ok"`. |
| `input_blob_hex` | string | The on-disk blob bytes (hex). |
| `input_root_key_hex` | string | The correct `root_key` (hex). |
| `input_account_number_hash_hex` | string | The correct `account_number_hash` (hex). |
| `expected_plaintext_canonical_utf8_hex` | string | The exact canonical-JSON plaintext bytes the decoder recovers (hex). |
| `expected_prefs_view` | object | The parsed `PrefsView` shape — implementations decode the JSON and assert structural equality. Phase 1: `{"screen_capture_protection": <bool>}`. |

### `case_kind = "decrypt_aead_fail"` — pin the non-distinguishing failure surface

| Field | Type | Description |
|---|---|---|
| `vector_id` | string | Stable identifier. |
| `description` | string | What the case verifies. |
| `case_kind` | string | `"decrypt_aead_fail"`. |
| `input_blob_hex` | string | The blob bytes the decoder attempts (hex). May be a valid blob from a different vault (cross-vault attack scenario) OR a tampered blob (one byte flipped, etc.). |
| `input_root_key_hex` | string | The `root_key` (hex) passed to the decoder. For cross-vault attacks, this is vault-B's `root_key` while `input_blob_hex` is vault-A's. For tampered, this is the original blob's `root_key`. |
| `input_account_number_hash_hex` | string | The `account_number_hash` (hex) passed to the decoder. For cross-vault attacks, this is vault-B's hash. |
| `expected_kind` | string | Always `"aead_fail"`. The decoder MUST report a non-distinguishing AEAD failure (`CannotDecrypt` in the Rust reference; "wrong key or tampered" semantically — implementations MUST NOT distinguish key-mismatch from AAD-mismatch from ciphertext-tampering at the caller-visible surface per spec § 6.4). The CTX commitment construction makes the four failure arms (wrong key, wrong AAD, tampered ciphertext, tampered CTX tag) computationally indistinguishable. |

## How to consume

A port runs each case based on `case_kind`:

- **`encrypt`**: decode every `input_*_hex`, build a `PrefsView` from the implied plaintext shape, invoke the port's `encode_prefs_blob_raw` equivalent with the supplied nonce, hex-encode the output, compare byte-equal to `expected_blob_hex`. Any non-equal result is a stop-the-world bug (per repo `README.md` "Cross-implementation discipline").
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
