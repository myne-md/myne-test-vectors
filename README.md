# myne-test-vectors

Language-agnostic **JSON test vectors** for the [Myne](https://github.com/myne-md/myne) cryptographic protocol.

For the protocol specification, see [`docs/spec/protocol.md`](https://github.com/myne-md/myne/blob/main/docs/spec/protocol.md) in the main repository.

## Status

**Phase 1, nine of eleven planned v0.1 categories landed.** Each landed category has a JSON-Schema-validated `cases.json` plus a domain-semantic `schema.md`. The remaining two are planned but currently have no Plane task tracking them — they land when their parent feature stabilizes and a Plane work item is created. The canonical machine-readable contract lives at [`schema/v0.1.json`](schema/v0.1.json) and dispatches per-category sub-schemas under [`schema/v0.1/`](schema/v0.1/).

### Landed (v0.1)

- **[`vectors/phrase_normalize/`](vectors/phrase_normalize/)** — `protocol::phrase::normalize_user_phrase` NFKD + ASCII-lowercase + whitespace pipeline.
- **[`vectors/vault-prefs/`](vectors/vault-prefs/)** — `MYP1` encrypted-preferences blob codec (`encode_prefs_blob_raw` / `decode_prefs_blob`), with encrypt + decrypt-success + non-distinguishing AEAD-failure cases.
- **[`vectors/vault_meta_rewrite/`](vectors/vault_meta_rewrite/)** — `rewrite_vault_meta_password_side` password-side rewrite producing the 345-byte `MYV1` blob.
- **[`vectors/hkdf-sha256/`](vectors/hkdf-sha256/)** — `derive_subkey` canonical HKDF-SHA-256 helper exercising every registered `SubkeyRole`.
- **[`vectors/vault-meta/`](vectors/vault-meta/)** — `serialize_vault_meta` + `parse_vault_meta` create-time `MYV1` encoder/decoder pair: 345-byte round-trip + structural-parse-reject cases (magic / reserved tamper). Distinct from `vault_meta_rewrite` (which pins only the password-side rewrite).
- **[`vectors/account-number/`](vectors/account-number/)** — `parse_account_number` Crockford-base32 + Damm-32 + liberal-parsing normalization: parse permutations (canonical/display/lowercase/whitespace/prefix-optional) + strict-reject (non-Crockford char, wrong length) + Damm-32 single-substitution + adjacent-transposition catches.
- **[`vectors/myn-blob/`](vectors/myn-blob/)** — `encode_new_note_blob_raw` + `decode_note` `MY01` per-note blob codec across 10 B / 1 KB / 100 KB plaintext sizes + decrypt round-trip + ciphertext-tamper reject. `content_type` is hardcoded `text/markdown` in v0.1.
- **[`vectors/ctx-aead/`](vectors/ctx-aead/)** — `aead_seal_ctx` + `aead_open_ctx` (XChaCha20-Poly1305 + CTX commitment outer per Chan & Rogaway 2022) + every pub `aad_*` builder's canonical byte assembly (`aad_password_wrap`, `aad_recovery_wrap`, `aad_vault_items_key_wrap`, `aad_note_item_key_wrap`, `aad_note_content_encrypt`, `aad_prefs_blob` — all 6 pub builders pinned). Note: this category was tracked in the prior planned list as `xchacha20-poly1305-ctx`; landed under `ctx-aead` (same scope).
- **[`vectors/argon2id/`](vectors/argon2id/)** — `derive_password_wrap_key` + `derive_recovery_wrap_key_from_normalized` at canonical Phase 1 parameters (m=256 MiB, t=3, p=2, v=0x13). Password + recovery paths × {short ASCII, long ASCII, unicode}. **Slow tests** (~3s per case at canonical params; ~18s total category runtime).

### Planned (no Plane task yet — land when a Plane work item is created)

- `vectors/bip39-recovery/` — 24-word phrase → `recovery_wrap_key` derivation through Argon2id (NOT through BIP-39's PBKDF2 step), including NFKD normalization edge cases. Composes the existing `phrase_normalize` + `argon2id` categories' pipeline into end-to-end vectors.
- `vectors/aad-construction/` — Canonical AAD byte assembly for every AEAD call site. **Note:** the 6 pub `aad_*` builders are ALREADY pinned individually inside the `ctx-aead` category via its `aad_canonical_bytes` case kind. A future dedicated `aad-construction/` category would expand coverage to per-call-site framing (the full AEAD call's bound metadata, not just the builder output), but the current ctx-aead coverage closes the cross-impl gap for the builder layer.

Each category pins the protocol version (`v0.1` for the initial set) so that future major-version vectors (`v005` and beyond) can coexist without ambiguity.

## Schema layout

```
schema/
├── v0.1.json                          ← root schema (top-level shape + oneOf dispatch on `schema` field)
└── v0.1/
    ├── phrase_normalize.json          ← per-category case shape
    ├── vault-prefs.json
    ├── vault_meta_rewrite.json
    ├── hkdf-sha256.json
    ├── vault-meta.json
    ├── account-number.json
    ├── myn-blob.json
    ├── ctx-aead.json
    └── argon2id.json
```

Adding a new category requires landing its sub-schema file under `schema/v0.1/` and extending both the `schema` enum and the `oneOf` branches in the root schema in lockstep.

## Validating vectors locally

Schema validation is currently a developer-side step — CI integration is deferred. Run from this repository's root:

```bash
npx -y ajv-cli@5 validate \
  -s schema/v0.1.json \
  -r schema/v0.1/phrase_normalize.json \
  -r schema/v0.1/vault-prefs.json \
  -r schema/v0.1/vault_meta_rewrite.json \
  -r schema/v0.1/hkdf-sha256.json \
  -r schema/v0.1/vault-meta.json \
  -r schema/v0.1/account-number.json \
  -r schema/v0.1/myn-blob.json \
  -r schema/v0.1/ctx-aead.json \
  -r schema/v0.1/argon2id.json \
  -d "vectors/phrase_normalize/v0.1/cases.json" \
  -d "vectors/vault-prefs/v0.1/cases.json" \
  -d "vectors/vault_meta_rewrite/v0.1/cases.json" \
  -d "vectors/hkdf-sha256/v0.1/cases.json" \
  -d "vectors/vault-meta/v0.1/cases.json" \
  -d "vectors/account-number/v0.1/cases.json" \
  -d "vectors/myn-blob/v0.1/cases.json" \
  -d "vectors/ctx-aead/v0.1/cases.json" \
  -d "vectors/argon2id/v0.1/cases.json" \
  --spec=draft2020
```

Every `cases.json` must validate clean. Byte-level conformance against the live Rust impl is verified separately via the integration tests in the main repo at `protocol/tests/vectors.rs` (`cargo test -p protocol --test vectors`).

## Why language-agnostic JSON

Any independent implementation of the Myne protocol — in Rust, Go, Swift, Kotlin, JavaScript, or anything else — must produce bit-identical outputs from these inputs. A reviewer or contributor verifying a port should not need to read the reference Rust implementation; running the test runner of their language against this repository should be enough.

## License

[CC0 1.0](LICENSE) — public domain. Test vectors are reference material for independent implementations; they should be maximally reusable.
