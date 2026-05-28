# myne-test-vectors

Language-agnostic **JSON test vectors** for the [Myne](https://github.com/myne-md/myne) cryptographic protocol.

For the protocol specification, see [`docs/spec/protocol.md`](https://github.com/myne-md/myne/blob/main/docs/spec/protocol.md) in the main repository.

## Status

**Phase 1, four of eleven planned v0.1 categories landed.** Each landed category has a JSON-Schema-validated `cases.json` plus a domain-semantic `schema.md`. The remaining seven are planned — five next-actionable (each its own Plane task) plus two deferred (blocked on their parent feature stabilizing). The canonical machine-readable contract lives at [`schema/v0.1.json`](schema/v0.1.json) and dispatches per-category sub-schemas under [`schema/v0.1/`](schema/v0.1/).

### Landed (v0.1)

- **[`vectors/phrase_normalize/`](vectors/phrase_normalize/)** — `protocol::phrase::normalize_user_phrase` NFKD + ASCII-lowercase + whitespace pipeline.
- **[`vectors/vault-prefs/`](vectors/vault-prefs/)** — `MYP1` encrypted-preferences blob codec (`encode_prefs_blob_raw` / `decode_prefs_blob`), with encrypt + decrypt-success + non-distinguishing AEAD-failure cases.
- **[`vectors/vault_meta_rewrite/`](vectors/vault_meta_rewrite/)** — `rewrite_vault_meta_password_side` password-side rewrite producing the 345-byte `MYV1` blob.
- **[`vectors/hkdf-sha256/`](vectors/hkdf-sha256/)** — `derive_subkey` canonical HKDF-SHA-256 helper exercising every registered `SubkeyRole`.

### Planned (next-actionable — each its own Plane task)

- `vectors/argon2id/` — KDF outputs for fixed inputs at the canonical Phase 1 parameters (`m=256 MiB, t=3, p=2`, Argon2 1.3) plus parameter-block AAD binding cases.
- `vectors/xchacha20-poly1305-ctx/` — AEAD encrypt/decrypt cases including the CTX commitment outer hash (Chan & Rogaway 2022) at the on-disk tag width of 32 bytes.
- `vectors/bip39-recovery/` — 24-word phrase → `recovery_wrap_key` derivation through Argon2id (NOT through BIP-39's PBKDF2 step), including NFKD normalization edge cases.
- `vectors/account-number/` — Crockford base32 encoding, Damm-32 check digit generation and validation, liberal-parsing normalization (case, hyphens, the `MY-` prefix).
- `vectors/aad-construction/` — Canonical AAD byte assembly for every AEAD call site, ensuring no implementation can silently disagree on the bound metadata.

### Planned (deferred — blocked on parent feature stabilizing)

- `vectors/vault-meta/` — Full `MYV1` create-time byte strings + decoded fields + tampering rejection cases. Distinct from the landed `vectors/vault_meta_rewrite/` (which only pins the password-side rewrite path); the create-time encoder vector spec lands when its parent-feature byte layout is final.
- `vectors/myn-blob/` — Full `MY01` per-note byte strings + content-type variations. Blocked on `protocol/blob.rs` create-time encoder stabilizing in the main repo.

Each category pins the protocol version (`v0.1` for the initial set) so that future major-version vectors (`v005` and beyond) can coexist without ambiguity.

## Schema layout

```
schema/
├── v0.1.json                          ← root schema (top-level shape + oneOf dispatch on `schema` field)
└── v0.1/
    ├── phrase_normalize.json          ← per-category case shape
    ├── vault-prefs.json
    ├── vault_meta_rewrite.json
    └── hkdf-sha256.json
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
  -d "vectors/phrase_normalize/v0.1/cases.json" \
  -d "vectors/vault-prefs/v0.1/cases.json" \
  -d "vectors/vault_meta_rewrite/v0.1/cases.json" \
  -d "vectors/hkdf-sha256/v0.1/cases.json" \
  --spec=draft2020
```

Every `cases.json` must validate clean. Byte-level conformance against the live Rust impl is verified separately via the integration tests in the main repo at `protocol/tests/vectors.rs` (`cargo test -p protocol --test vectors`).

## Why language-agnostic JSON

Any independent implementation of the Myne protocol — in Rust, Go, Swift, Kotlin, JavaScript, or anything else — must produce bit-identical outputs from these inputs. A reviewer or contributor verifying a port should not need to read the reference Rust implementation; running the test runner of their language against this repository should be enough.

## License

[CC0 1.0](LICENSE) — public domain. Test vectors are reference material for independent implementations; they should be maximally reusable.
