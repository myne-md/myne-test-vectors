# `vault_meta_rewrite` vector schema

Pins the byte mapping of `protocol::rewrite_vault_meta_password_side` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)). The function rewrites the password side of an existing `vault.meta` while byte-preserving the recovery side and the `items_key` block — the load-bearing primitive behind the post-recovery password commit (`vault_complete_recovery` / A8f). Reference implementation: `protocol/src/vault_meta.rs::rewrite_vault_meta_password_side`. Spec: `docs/spec/protocol.md` § 9.3 (on-disk layout) + `docs/spec/recovery-flow.md` § 4.2 (atomicity) + § 4.4 (recovery-side preservation).

## Pipeline (as fixed by the reference implementation)

For each input set:

1. **Derive new `password_wrap_key`** via Argon2id with input `new_password` (UTF-8 bytes), `fresh_salt`, `existing.account_number_hash`, and `kdf_params`. Salt prefix is `"myne.v0.1 argon2id-password|"` per spec § 3.1.
2. **Build the password-wrap AAD** via `aad_password_wrap(kdf_params)`.
3. **AEAD-seal** `root_key` under the new wrap-key + `fresh_nonce` + AAD using XChaCha20-Poly1305 with the CTX (Chan & Rogaway 2022) outer commitment per spec § 5.1. Output is exactly 64 bytes (32 ciphertext + 32 commitment) for a 32-byte plaintext.
4. **Assemble a fresh `VaultMeta`** with password-side fresh (`random_salt_password = fresh_salt`, `nonce_password_wrap = fresh_nonce`, `enc_root_key_password = sealed`, `kdf_params = new params`), `account_number_hash` byte-copied from `existing`, recovery-side fields byte-copied from `existing`, `items_key_id` + `items_key_nonce` + `enc_items_key` byte-copied from `existing`.
5. **Serialize** via the canonical encoder (spec § 9.3 fixed-offset layout, total 345 bytes).

The function is **pure and deterministic** — same inputs produce identical output bytes; no internal CSPRNG, no clock. Callers (the `vault_recover` Tauri command) generate `fresh_salt` + `fresh_nonce` via OS CSPRNG before invoking; the function itself is byte-stable.

## File layout

```
vectors/vault_meta_rewrite/
├── schema.md              ← this file
└── v0.1/
    └── cases.json         ← array of input → expected_output cases
```

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/vault_meta_rewrite.json`](../../schema/v0.1/vault_meta_rewrite.json) (per-case shape; single `case_kind`, no discriminator). Run the repository's documented `ajv-cli validate` command (see the root `README.md`) to verify any `cases.json` against the canonical schema. The sub-schema is authoritative — this document covers only the domain-specific semantics.

Each case pins one rewrite: an `input_existing_vault_meta_hex` (345-byte `MYV1` blob) plus the new password material (`input_new_password_utf8_hex` + `input_fresh_salt_hex` + `input_fresh_nonce_hex`) plus the unwrapped `input_root_key_hex` produces a fresh `expected_vault_meta_hex` (also 345 bytes). The informational `informational_kdf_params_from_existing` object mirrors the KDF parameters carried inside the existing blob's offset 22 so reviewers can verify intermediate state without re-decoding the existing-meta hex; the helper reads them from the parsed input, NOT from this informational field.

## How to consume

A port runs each case as: decode every `*_hex` input field (the KDF parameters used by the helper are read from `input_existing_vault_meta_hex` itself, not from `informational_kdf_params_from_existing` — the informational object mirrors them for reviewer convenience), invoke the port's `rewrite_vault_meta_password_side` equivalent, hex-encode the output, compare byte-equal to `expected_vault_meta_hex`. Any non-equal result is a stop-the-world bug (per repo `README.md` "Cross-implementation discipline").

## Why hex, not JSON-native UTF-8 strings or numeric arrays

JSON natively supports UTF-8 strings via `\uXXXX` escapes, but escape behavior varies subtly across language libraries. JSON also supports numeric arrays for byte sequences, but byte-array parsing varies and is awkward to diff. Hex-encoding the raw bytes eliminates parser-side transformations: bytes that go in are exactly the bytes the reference implementation saw.

## Invariants the vectors pin

For the canonical case in `cases.json`:

- **`account_number_hash` byte-preserved.** Output `[6..22]` equals `existing[6..22]`.
- **Recovery-side fields byte-preserved (recovery-flow.md § 4.4).** Output `[49..65]` (random_salt_recovery), `[153..177]` (nonce_recovery_wrap), `[177..241]` (enc_root_key_recovery) equal the corresponding `existing` slices.
- **`items_key_id` + `items_key_nonce` + `enc_items_key` byte-preserved.** Output `[241..345]` equals `existing[241..345]`. The AEAD AAD on `enc_items_key` binds `items_key_id`; binary copy is the only safe path.
- **Password-side fresh.** Output `[33..49]` equals `fresh_salt`; output `[65..89]` equals `fresh_nonce`. Output `[89..153]` is the new AEAD-sealed `enc_root_key_password` derived from the new inputs.
- **KDF parameter block matches input.** Output `[22..33]` equals the canonical 11-byte serialization of the KDF params parsed from `input_existing_vault_meta_hex` (per protocol § 4.3) — the helper preserves these from the input meta, not from a separate parameter; the `informational_kdf_params_from_existing` object mirrors the same bytes for reviewer convenience.
- **Magic / version / reserved.** Output `[0..4]` = `MYV1`, `[4]` = `0x01`, `[5]` = `0x00`.

## Regeneration

If the reference implementation changes (must not happen byte-stably without a major-version event), regenerate this vector via:

```bash
cargo test -p protocol --lib emit_vector_hex_for_paste -- --ignored
cat /tmp/myne_vault_meta_rewrite_vector.txt
# paste hex outputs into cases.json
```

The regenerator test lives in `protocol/src/vault_meta.rs::tests::emit_vector_hex_for_paste`. It is `#[ignore]`-gated so it never runs in default `cargo test` invocations.

## What this does NOT cover

- The full `vault.meta` serialization vectors (`vectors/vault-meta/`, planned). Those pin the create-time encoder's byte mapping; this directory pins only the password-side rewrite path.
- The post-rewrite atomic-write step (tempfile + fsync + rename). That happens at the Tauri-layer in `vault_complete_recovery`; this primitive is filesystem-agnostic.
- Vault-side AEAD verification through the rewritten password wrap. That is covered indirectly by the in-crate test `rewrite_password_side_round_trips_root_key` (re-derives wrap-key, AEAD-decrypts, asserts recovered root_key matches input).
