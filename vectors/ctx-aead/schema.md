# `ctx-aead` vector schema

Pins the byte mapping of `protocol::aead_seal_ctx` + `protocol::aead_open_ctx` (XChaCha20-Poly1305 with CTX commitment outer per Chan & Rogaway 2022) AND the canonical byte assembly of every pub `aad_*` builder in `protocol/src/aead.rs`. Together these capture the entire AEAD layer's cross-implementation contract.

**Reference implementation:** `protocol/src/aead.rs::{aead_seal_ctx, aead_open_ctx, aad_password_wrap, aad_recovery_wrap, aad_vault_items_key_wrap, aad_note_item_key_wrap, aad_note_content_encrypt, aad_prefs_blob, aad_search_index_blob, aad_workspace_blob}`.
**Spec:** `docs/spec/protocol.md` § 5.1 (AEAD construction — XChaCha20-Poly1305 + CTX commitment) + § 5.2 (AAD byte assembly) + § 6.4 (non-distinguishing failure surface).

## Pipeline (as fixed by spec § 5.1)

**AEAD seal** (`aead_seal_ctx`):
1. **XChaCha20-Poly1305 seal** of `plaintext` under (`key`, `nonce`, `aad`) producing `ciphertext` (plaintext.len() bytes) + 16-byte Poly1305 tag.
2. **CTX commitment**: compute `SHA-256(key || nonce || u64-LE(aad.len()) || aad || poly1305_tag)` → 32-byte commitment. This is the OUTER commitment that compensates for Poly1305's known commitment weakness against multi-key attacks.
3. **Return** `ciphertext || ctx_tag` (ciphertext.len() + 32 bytes). The 16-byte Poly1305 tag is intentionally discarded — only the 32-byte CTX commitment ships on disk.

**AEAD open** (`aead_open_ctx`): the inverse — recompute Poly1305 tag from `(key, nonce, ciphertext, aad)`, recompute CTX commitment, constant-time compare against the stored 32-byte tag. Any mismatch surfaces non-distinguishing `Err(ProtocolError::AeadFailure)` per spec § 6.4. Decrypt the ciphertext only after the commitment check passes.

**AAD builders** (`aad_*` functions): pure functions producing the canonical AAD byte sequence for each AEAD call site per spec § 5.2. Each builder follows the `"myne.v0.1|"` prefix + version byte + role marker + role-specific bound bytes pattern. Eight builders cover every AEAD invocation site in the protocol crate:

| Builder | Bound bytes | Spec § | Notes |
|---|---|---|---|
| `aad_password_wrap(kdf_params)` | KDF parameter block (11 B) | §5.2 | Used for `enc_root_key_password` in `vault.meta`. |
| `aad_recovery_wrap(kdf_params)` | KDF parameter block (11 B) | §5.2 | Used for `enc_root_key_recovery`. Differs from `aad_password_wrap` by the role byte ('rk' vs 'pw'). |
| `aad_vault_items_key_wrap(items_key_id)` | items_key_id (16 B) | §5.2 | Singleton itemsKey wrap — no per-note UUID. |
| `aad_note_item_key_wrap(items_key_id, note_uuid)` | items_key_id (16 B) + note_uuid (16 B) | §5.2 | Per-note item_key wrap. One-character name difference from `aad_vault_items_key_wrap` (`items_key` vs `item_key`) is a documented type-confusion hazard class. |
| `aad_note_content_encrypt(items_key_id, note_uuid)` | items_key_id + note_uuid + length-prefixed `"text/markdown"` | §5.2 | Content encryption — content_type is HARDCODED to `text/markdown`. |
| `aad_prefs_blob(account_number_hash)` | account_number_hash (16 B) | §9.5.3 | `vault.prefs` AEAD AAD. Binds the blob to its specific vault. |
| `aad_search_index_blob(account_number_hash)` | account_number_hash (16 B) | §9.6.3 | `vault.search` AEAD AAD (MYND-29). Same shape as `aad_prefs_blob`, distinct 17-byte role `search-index-blob`. |
| `aad_workspace_blob(account_number_hash)` | account_number_hash (16 B) | §9.7.3 | `vault.workspace` AEAD AAD (MYND-254). Same shape, distinct 14-byte role `workspace-blob` — NOT the HKDF KEK role `session-state-key`. |

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/ctx-aead.json`](../../schema/v0.1/ctx-aead.json) (per-case shape with a 4-way `case_kind` discriminator: `encrypt`, `decrypt_ok`, `decrypt_aead_fail`, `aad_canonical_bytes`). Run the repository's documented `ajv-cli validate` command (see the root `README.md`) to verify any `cases.json` against the canonical schema. The sub-schema is authoritative — this document covers only the domain-specific semantics.

Four case kinds carry different field shapes:

- **`encrypt`** — pin the AEAD encoder's byte output at the primitive level. Inputs: raw `key` (32 B), `nonce` (24 B), `aad` (variable), `plaintext` (variable). Output: `expected_sealed_hex` = ciphertext + 32-byte CTX tag concatenated.
- **`decrypt_ok`** — pin the decoder's successful round-trip. Inputs: same key+nonce+aad+sealed. Output: `expected_plaintext_hex` (the recovered plaintext).
- **`decrypt_aead_fail`** — pin the non-distinguishing AEAD failure. Inputs as above but with one of {key, nonce, aad, sealed} tampered. Output: `expected_kind: "aead_fail"`. Implementations MUST collapse all rejection causes (wrong key / wrong nonce / wrong AAD / tampered ciphertext / tampered tag) to their port's equivalent of `Err(ProtocolError::AeadFailure)`.
- **`aad_canonical_bytes`** — pin a specific pub `aad_*` builder's canonical output bytes. Input: `input_builder_name` (enum) + the builder-specific parameter fields. Output: `expected_aad_bytes_hex` (the canonical AAD assembly). No AEAD invocation — this case kind tests the AAD-assembly layer in isolation.

## How to consume

A port runs each case based on `case_kind`:

- **`encrypt`** / **`decrypt_ok`** / **`decrypt_aead_fail`**: standard primitive byte-level assertions against `aead_seal_ctx` / `aead_open_ctx`.
- **`aad_canonical_bytes`**: dispatch on `input_builder_name` to the matching builder, decode the builder's required input fields, call the builder, compare output byte-equal to `expected_aad_bytes_hex`. Ports MUST cover all 8 builders (the Rust integration runner enforces a `HashSet`-tracked completeness check per category).

## Invariants the vectors pin

For the 18 cases in `cases.json`:

- **Every pub `aad_*` builder gets at least one canonical-bytes pin.** The 8 `aad_canonical_bytes` cases collectively name each builder by `input_builder_name`. Drift in any builder's byte assembly surfaces immediately on regeneration.
- **Primitive AEAD byte output at empty-AAD and content-AAD scopes.** The 2 `encrypt` cases cover (a) bare-primitive bytes with no AAD, (b) realistic AAD shape (58-byte `aad_note_content_encrypt` output) — same primitive, different AAD context.
- **Round-trip property.** `decrypt-ok-medium` proves the decoder recovers the canonical plaintext from the medium encrypt case's sealed bytes.
- **Non-distinguishing AEAD failure surface — two distinct tamper vectors.** `decrypt-tag-tamper` flips one byte of the 32-byte CTX commitment tag; `decrypt-aad-tamper` passes a swapped AAD (the `aad_prefs_blob` bytes instead of the `aad_note_content_encrypt` bytes the encoder used). Both surface `Err(ProtocolError::AeadFailure)` — ports MUST collapse cross-vault, wrong-AAD, tampered-ciphertext, and tampered-tag to the same non-distinguishing variant.
- **The `items_key` / `item_key` naming-discipline hazard is pinned.** The `aad-vault-items-key-wrap` (singleton itemsKey, no per-note UUID) and `aad-note-item-key-wrap` (per-note item_key, with note_uuid) cases sit adjacent in `cases.json` — a side-by-side byte diff documents the one-character name difference + corresponding AAD shape difference. Per `protocol/CLAUDE.md` AAD-helper naming discipline rule.

## Regeneration

If the reference implementation changes (must not happen byte-stably without a major-version event), regenerate this vector set via:

```bash
cargo test -p protocol --lib emit_ctx_aead_vectors_for_paste -- --ignored
cp /tmp/myne_ctx_aead_vectors.json vectors/ctx-aead/v0.1/cases.json
```

The regenerator test lives in `protocol/src/aead.rs::tests::emit_ctx_aead_vectors_for_paste`. `#[ignore]`-gated. The generator includes its own round-trip + tamper-reject sanity asserts before emitting — a future spec change that broke either property would surface inside the generator rather than landing a bad cases.json.

## What this does NOT cover

- **AAD builder fixture coverage at multiple parameter values.** Each builder gets exactly one canonical-bytes pin at fixed test inputs. A future expansion could parameter-sweep (e.g., 3 `kdf_params` values × `aad_password_wrap`) if cross-impl drift is observed.
- **Per-call AEAD seal/open invocation matrix.** The 2 encrypt + 1 decrypt_ok cover the primitive's byte-level contract; the higher-level consumers (`prefs.rs`, `vault_meta.rs::rewrite`, `blob.rs`) get their own dedicated vector categories (`vault-prefs`, `vault_meta_rewrite`, `myn-blob`) that exercise the full assembly + AEAD pipeline.
- **CTX commitment construction at multiple plaintext sizes.** The 256-byte medium case is representative; the `vectors/myn-blob/v0.1/` set covers 10 B / 1 KB / 100 KB at the consumer layer where the CTX commitment is also exercised. Adding a 0-byte plaintext case here would document the bare-tag corner; deferred unless observed.
- **Multi-key key-confusion / commitment-attack scenarios.** The CTX commitment compensates for Poly1305's commitment weakness against multi-key attacks; the vectors here exercise byte-equality, not the commitment property itself. A future research-grade test vector set could pin (key1, key2) collisions; out of v0.1 scope.
