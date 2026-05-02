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

Top-level object:

| Field | Type | Description |
|---|---|---|
| `schema` | string | Always `"vault_meta_rewrite"` for this directory. |
| `protocol_version` | string | `"v0.1"` for v0.1 vectors. |
| `spec_section` | string | Anchors the canonical pipeline definition (`"protocol.md §9.3 + recovery-flow.md §4.2 §4.4"`). |
| `description` | string | Human-readable summary of what is pinned. |
| `cases` | array | One element per test case (see below). |

Each `cases[i]` element:

| Field | Type | Description |
|---|---|---|
| `vector_id` | string | Stable identifier (`"vault_meta_rewrite/v0.1/<slug>"`). Immutable once shipped. |
| `description` | string | What the case verifies. |
| `input_existing_vault_meta_hex` | string | Lowercase hex (no `0x` prefix) of the 345-byte serialized `vault.meta` that exists prior to the password rewrite. |
| `input_root_key_hex` | string | 32-byte `root_key` (hex) — the unwrapped key the rewrite re-seals under the new password. |
| `input_new_password_utf8_hex` | string | UTF-8 bytes (hex) of the new master password. |
| `input_fresh_salt_hex` | string | 16-byte fresh Argon2id salt (hex). Production callers source this from OS CSPRNG; vectors pin a constant for reproducibility. |
| `input_fresh_nonce_hex` | string | 24-byte fresh XChaCha20-Poly1305 nonce (hex). Production callers source this from OS CSPRNG. |
| `informational_kdf_params_from_existing` | object | The KDF parameters carried inside `input_existing_vault_meta_hex` at offset 22. Informational-only — the helper reads them from the parsed input, NOT from a separate parameter. Mirrored here so reviewers don't have to decode the existing-meta hex. Fields: `m_kib` (u32), `t` (u32), `p` (u8), `argon2_version_hex` (one-byte hex, `"13"` for Argon2 v1.3). |
| `expected_output_hex` | string | Lowercase hex of the 345-byte rewritten `vault.meta`. |

## How to consume

A port runs each case as: decode every `*_hex` input field, parse the `kdf_params` object into the port's KDF-params type, invoke the port's `rewrite_vault_meta_password_side` equivalent, hex-encode the output, compare byte-equal to `expected_output_hex`. Any non-equal result is a stop-the-world bug (per repo `README.md` "Cross-implementation discipline").

## Why hex, not JSON-native UTF-8 strings or numeric arrays

JSON natively supports UTF-8 strings via `\uXXXX` escapes, but escape behavior varies subtly across language libraries. JSON also supports numeric arrays for byte sequences, but byte-array parsing varies and is awkward to diff. Hex-encoding the raw bytes eliminates parser-side transformations: bytes that go in are exactly the bytes the reference implementation saw.

## Invariants the vectors pin

For the canonical case in `cases.json`:

- **`account_number_hash` byte-preserved.** Output `[6..22]` equals `existing[6..22]`.
- **Recovery-side fields byte-preserved (recovery-flow.md § 4.4).** Output `[49..65]` (random_salt_recovery), `[153..177]` (nonce_recovery_wrap), `[177..241]` (enc_root_key_recovery) equal the corresponding `existing` slices.
- **`items_key_id` + `items_key_nonce` + `enc_items_key` byte-preserved.** Output `[241..345]` equals `existing[241..345]`. The AEAD AAD on `enc_items_key` binds `items_key_id`; binary copy is the only safe path.
- **Password-side fresh.** Output `[33..49]` equals `fresh_salt`; output `[65..89]` equals `fresh_nonce`. Output `[89..153]` is the new AEAD-sealed `enc_root_key_password` derived from the new inputs.
- **KDF parameter block matches input.** Output `[22..33]` equals the canonical 11-byte serialization of `input_kdf_params` (per protocol § 4.3).
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
