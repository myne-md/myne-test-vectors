# `vault-quickunlock` vector schema

Pins the byte mapping of `protocol::encode_quick_unlock_blob_raw` + `protocol::decode_quick_unlock_blob` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)). The codec emits and consumes `vault.quickunlock` — the v0.1 **device-local credential** blob (`MYQ1` magic; MYND-329) holding a **third wrapped copy of `root_key`**, so a re-lock can be reversed with a fast second credential (Touch ID / Windows Hello / a PIN) instead of the full master-password Argon2id path.

**Reference implementation:** `protocol/src/quick_unlock_blob.rs::{encode_quick_unlock_blob_raw, decode_quick_unlock_blob, parse_quick_unlock_header}`.
**Spec:** `docs/spec/protocol.md` § 9.14 (`vault.quickunlock` outer envelope + fixed 64-byte ciphertext + AAD + provider model) + § 5.2 / § 9.14.3 `AAD_quick_unlock_blob` formula (the AAD role `"quick-unlock-blob"`).

## No HKDF subkey role — the one § 9 blob without one

Every other § 9 blob seals under `DeriveSubkey(root_key, …)`. `vault.quickunlock` does **not**: that would be circular, since the blob *is* what protects `root_key`. The 32-byte `wrap_key` is supplied by an **external provider** (macOS Keychain biometric secret, or a PIN run through Argon2id with the salt domain `"myne.v0.1 argon2id-quickunlock-pin|"`), and the codec is **provider-agnostic** — it only ever sees a symmetric `wrap_key`. So:

- there is **no `hkdf-sha256` role** for quick-unlock (do not look for one);
- the vectors supply `wrap_key` **directly** as a fixed fixture;
- the PIN→`wrap_key` Argon2id derivation is pinned **separately** in the `argon2id` category (`input_path: "quickunlock-pin"`), not here.

The only load-bearing role string is the **AAD role** `"quick-unlock-blob"` (17 ASCII bytes, length-prefixed), pinned independently in `ctx-aead` (`aad_canonical_bytes`) and decomposed in `aad-construction`.

## `provider_kind` is AAD-bound

`provider_kind` (0x01 biometric / 0x02 PIN) lives in the public header at offset 5 **and** is bound into the AAD via `aad_quick_unlock_blob(account_number_hash, provider_kind)`. A biometric-minted blob therefore cannot be reinterpreted as a PIN blob: flipping the byte from 0x01 to 0x02 passes the structural "is this a registered provider" check but fails AEAD verification (a `decrypt_aead_fail`, not a `parse_fail`). `pin_salt` (offset 6, 16 bytes) is **not** AAD-bound — it is public so a PIN caller can read it back (via `parse_quick_unlock_header`) to derive `wrap_key` before decode.

## Pipeline (as fixed by the reference implementation)

**Encrypt side** (`encode_quick_unlock_blob_raw`):

1. **Reject an unregistered `provider_kind`** (only 0x01 / 0x02) before anything else.
2. **Build the AAD** via `aad_quick_unlock_blob(account_number_hash, provider_kind)`: 47-byte concatenation of `"myne.v0.1|"` (10 B) + protocol_version `0x01` (1 B) + length-prefixed UTF-8 role `"quick-unlock-blob"` (2-B BE length `0x0011` + 17 ASCII bytes = 19 B) + `account_number_hash` (16 B) + `provider_kind` (1 B) per spec § 9.14.3.
3. **AEAD-seal** the 32-byte `root_key` plaintext under `wrap_key` + `nonce` + AAD using XChaCha20-Poly1305 with the CTX outer commitment (§ 5.1). Output is always `32 + 32 = 64` bytes.
4. **Assemble the outer envelope** per spec § 9.14.1 fixed-offset layout (total 114 bytes):
   - `[0..4]` magic `MYQ1`
   - `[4]` protocol_version `0x01`
   - `[5]` provider_kind (0x01 / 0x02)
   - `[6..22]` pin_salt (16 B; all-zero for biometric)
   - `[22..46]` nonce (24 B)
   - `[46..50]` ciphertext_len (big-endian uint32, always 64)
   - `[50..114]` ciphertext = sealed root_key (32) + 32-byte CTX tag

**Decrypt side** (`decode_quick_unlock_blob`): parse header, validate magic / protocol_version / provider_kind (all checked **before** any decryption), read pin_salt + nonce + ciphertext_len (reject `!= 64` → `CiphertextLenInvalid`), rebuild AAD from `account_number_hash` + the header's `provider_kind`, AEAD-decrypt. AEAD failure (wrong wrap_key / wrong AAD / tampered ciphertext / provider-flip) collapses to a single non-distinguishing `CannotDecrypt` (§ 6.4).

The encode side is **pure and deterministic** given the caller-supplied `nonce`. Production callers use the `encode_quick_unlock_blob` convenience wrapper that pulls a nonce from `getrandom`; vectors pin a constant nonce for reproducibility.

## File layout

```
vectors/vault-quickunlock/
├── schema.md              ← this file
└── v0.1/
    └── cases.json         ← array of 12 KAT cases (encrypt + decrypt + AEAD-fail + structural parse-fail)
```

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/vault-quickunlock.json`](../../schema/v0.1/vault-quickunlock.json) (per-case shape with a 4-way `case_kind` discriminator). Run the repository's documented `ajv-cli validate` command (see the root `README.md`) to verify any `cases.json`. The sub-schema is authoritative — this document covers only the domain-specific semantics.

Four case kinds carry different field shapes:

- **`encrypt`** — pins the encoder's byte output. Inputs: `input_wrap_key_hex` (32 B, supplied directly — NOT derived), `input_account_number_hash_hex` (16 B), `input_provider_kind` (integer 0x01/0x02), `input_pin_salt_hex` (16 B), `input_root_key_hex` (32 B, the wrapped secret), `input_nonce_hex` (24 B). Output: `expected_vault_quickunlock_blob_hex` (the 114-byte `MYQ1` blob).
- **`decrypt_ok`** — pins the decoder's successful round-trip. Inputs: `input_blob_hex` + correct `input_wrap_key_hex` + correct `input_account_number_hash_hex`. Output: `expected_root_key_hex` (the recovered 32-byte root_key).
- **`decrypt_aead_fail`** — pins the **non-distinguishing** failure surface. Inputs: `input_blob_hex` + (intentionally wrong/tampered) inputs. Output: `expected_kind: "aead_fail"`. Covers tag-tamper, cross-vault wrong `account_number_hash`, and the **provider_kind-flip** (0x01→0x02 on a valid biometric blob — AAD-bound, so it AEAD-fails). Fixtures carry the REAL wrap_key so a skipped check surfaces as a spurious `Ok`.
- **`parse_fail`** — pins a **distinguishable** pre-decryption structural reject. Inputs: `input_blob_hex` + `expected_error` (one of `magic_mismatch`, `protocol_version_mismatch`, `provider_kind_unsupported`, `ciphertext_len_invalid`, `header_too_short`, `ciphertext_truncated`). There is no `ciphertext_len_overflow` (the length is the fixed constant 64; an off-value is `ciphertext_len_invalid`).

## How to consume

A port runs each case based on `case_kind`:

- **`encrypt`**: invoke the port's `encode_quick_unlock_blob_raw` equivalent with the supplied wrap_key + account hash + provider_kind + pin_salt + root_key + nonce, hex-encode, compare byte-equal to `expected_vault_quickunlock_blob_hex`.
- **`decrypt_ok`**: invoke the decoder with the wrap_key + account hash, compare the recovered bytes to `expected_root_key_hex`.
- **`decrypt_aead_fail`**: invoke the decoder, assert a non-distinguishing AEAD failure. The port MUST NOT distinguish which arm fired (incl. the provider-flip).
- **`parse_fail`**: invoke the decoder, assert it returns the *specific* structural error named by `expected_error`, before any decryption. (wrap_key/account-hash are irrelevant — the reject precedes the AEAD stage.)

Any mismatch is a stop-the-world bug (per repo `README.md` "Cross-implementation discipline").

## Invariants the vectors pin

- **MYQ1 magic + version + provider_kind bytes.** Encoder output `[0..4]` = `MYQ1`, `[4]` = `0x01`, `[5]` = the supplied provider_kind. Decoder rejects bad magic / version / unregistered provider at parse time before AEAD.
- **Header byte layout.** `[6..22]` = pin_salt (16 B), `[22..46]` = nonce (24 B), `[46..50]` = ciphertext_len big-endian uint32 (= 64), `[50..114]` = ciphertext + 32-byte CTX tag.
- **Fixed 64-byte ciphertext.** The blob always wraps exactly one 32-byte root_key, so ciphertext_len is constant 64 — there is no variable-size cap; any other value is `CiphertextLenInvalid`.
- **AAD binds account identity AND provider_kind.** Cross-vault (wrong `account_number_hash`) and provider-flip (0x01→0x02) both fail AEAD — a credential from vault A cannot unwrap vault B's root_key, and a biometric credential cannot masquerade as a PIN one.
- **Non-distinguishing AEAD surface vs distinguishable structural surface.** The three `decrypt_aead_fail` cases MUST be indistinguishable from each other; the six `parse_fail` cases MUST each surface their specific structural reject (spec § 6.4 boundary policy).
- **Provider-flip is AEAD, not structural.** The `decrypt-provider-kind-flip` case (registered 0x02 value) reaches the AEAD stage and fails there; the `parse-fail-provider-kind-unsupported` case (unregistered 0x09 value) is rejected structurally before AEAD. The two MUST surface different outcomes.

## Regeneration

If the reference implementation changes (must not happen byte-stably without a major-version event), regenerate this vector via:

```bash
cargo test -p protocol --lib emit_quick_unlock_vectors_for_paste -- --ignored
cat /tmp/myne_vault_quickunlock_vectors.json
# copy the full JSON verbatim into cases.json
```

The regenerator test lives in `protocol/src/quick_unlock_blob.rs::tests::emit_quick_unlock_vectors_for_paste`. It is `#[ignore]`-gated so it never runs in default `cargo test` invocations.

## What this does NOT cover

- **The PIN→wrap_key Argon2id derivation.** The codec is provider-agnostic and never runs Argon2id; vectors supply `wrap_key` directly. The PIN derivation (`derive_quick_unlock_pin_wrap_key`, salt domain `"myne.v0.1 argon2id-quickunlock-pin|"`) is pinned separately in the `argon2id` category as `input_path: "quickunlock-pin"`.
- **Encode-side / unreachable error modes.** `QuickUnlockBlobError::PlaintextLenInvalid` (defensive — unreachable for a well-formed 64-byte ciphertext) and `RandomnessUnavailable` (OS CSPRNG failure in the non-`_raw` wrapper) cannot be expressed as a decode-a-blob case, so they are not vectored.
- **The wrap-key provider plumbing** (macOS Keychain / Windows Hello access, PIN entry policy, `MIN_PIN_LEN` host validation) — host-layer concerns verified by `src-tauri` integration tests, not cross-implementation vectors.
- **Schema evolution within v0.1.** A future provider_kind or protocol_version lands as a new case rather than mutating existing entries (immutable `vector_id`s).
