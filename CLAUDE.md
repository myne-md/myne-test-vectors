# myne-test-vectors — JSON test vectors for Myne v0.1 protocol

> Companion repository to [`myne`](https://github.com/myne-md/myne). Public, **CC0** (public domain) — vectors must remain maximally reusable for independent implementations in any language.

## Purpose

Language-agnostic JSON test vectors for the Myne cryptographic protocol. Any independent implementation (Rust, Go, Swift, Kotlin, JavaScript, etc.) MUST produce **bit-identical outputs** from these inputs. A reviewer or contributor verifying a port should not need to read the reference Rust implementation; running their language's test runner against this repository should be enough.

## Status

**Phase 1, canonical schema landed.** The `schema/v0.1.json` root schema + per-category sub-schemas under `schema/v0.1/` are the authoritative machine-readable contract. Four of the eleven planned v0.1 categories have landed (`phrase_normalize`, `vault-prefs`, `vault_meta_rewrite`, `hkdf-sha256`); five next-actionable plus two deferred fill in as Lane A primitives stabilize. CI integration is currently deferred — schema validation and byte-level integration run as developer-side steps. See main repo `agent tooling/features/test-vector-schema/` for the landing plan.

## Vector categories

### Landed (v0.1)

- `vectors/phrase_normalize/` — `protocol::phrase::normalize_user_phrase` NFKD + ASCII-lowercase + whitespace pipeline (spec §7.4).
- `vectors/vault-prefs/` — `MYP1` encrypted-preferences blob codec; encrypt + decrypt + non-distinguishing AEAD-failure case_kinds (spec §9.5 + §6.4).
- `vectors/vault_meta_rewrite/` — `MYV1` password-side rewrite producing the 345-byte vault.meta (spec §9.3 + recovery-flow §4.2/§4.4).
- `vectors/hkdf-sha256/` — Subkey derivations through `derive_subkey` exercising every registered `SubkeyRole` with `myne.v0.1 ` info-prefix (spec §3.2 + §3.4).

### Planned (next-actionable — each its own Plane task)

- `vectors/argon2id/` — KDF outputs at canonical Phase 1 params (`m=256 MiB, t=3, p=2`, Argon2 1.3) + parameter-block AAD binding cases
- `vectors/xchacha20-poly1305-ctx/` — AEAD encrypt/decrypt with CTX commitment outer (Chan & Rogaway 2022) at on-disk tag width of 32 bytes
- `vectors/bip39-recovery/` — 24-word phrase → `recovery_wrap_key` via Argon2id (NOT BIP-39's PBKDF2 step), including NFKD normalization edge cases
- `vectors/account-number/` — Crockford base32 encoding, Damm-32 check digit, liberal-parsing normalization (case, hyphens, `MY-` prefix)
- `vectors/aad-construction/` — Canonical AAD byte assembly for every AEAD call site

### Planned (deferred — blocked on parent feature stabilizing)

- `vectors/vault-meta/` — Full `MYV1` create-time byte strings + decoded fields + tampering rejection cases (distinct from the landed `vectors/vault_meta_rewrite/` which only pins password-side rewrite). Lands when the create-time encoder byte layout is final.
- `vectors/myn-blob/` — Full `MY01` per-note byte strings + content-type variations. Blocked on `protocol/blob.rs` create-time encoder stabilizing.

## JSON schema convention

Every vector file:
- Hex-encoded inputs (`key`, `nonce`, `aad`, `salt`, etc.) — lowercase, no `0x` prefix
- Hex-encoded expected outputs
- `spec_section` reference (e.g., `"protocol.md §5.1.1"`)
- `vector_id` (stable string, never reused)
- `protocol_version: "v0.1"` — pinned per vector so v005+ vectors can coexist without ambiguity

Field naming convention applied across every `cases.json`:
- Identity: `vector_id` (string, format `<category>/<version>/<descriptive-slug>`)
- Discriminator: `case_kind` (optional, present only when a category has more than one case shape)
- Inputs: `input_<semantic>_<encoding>` (e.g., `input_root_key_hex`, `input_plaintext_utf8_hex`)
- Expected outputs: `expected_<semantic>_<encoding>` (e.g., `expected_subkey_hex`, `expected_prefs_blob_hex`)
- Failure-mode expectation: `expected_kind` (string, e.g., `aead_fail`)
- Informational / intermediate: `informational_<semantic>[_<encoding>]` (suffix optional for structured objects)

The canonical machine-readable contract lives at `schema/v0.1.json` (top-level shape) and `schema/v0.1/<category>.json` (per-category case shape). The root schema dispatches via `oneOf` keyed on the `schema` field.

## v005 reservation

Future major-version vectors will live under `vectors/<cat>/v005/` subdirectories alongside v0.1 (`vectors/<cat>/v0.1/`), validated by a separate `schema/v005.json`. v005 magic bytes: `MY05` / `MYV5`; HKDF prefix: `"myne.v005 "`. v0.1 and v005 are **forward-incompatible by design** — they share this repo for distribution convenience, not for coexistence within a single client.

## Cross-implementation discipline

Independent ports (Phase 2 Go server, Phase 3 web/mobile clients) verify primitive-level compatibility by running these vectors through their language's runner. **A failed vector in any port is a stop-the-world bug** — either the port has drifted or the spec is wrong. Don't regenerate vectors to "fix" a failing port without reading why.

## Common commands

```bash
# Validate every landed cases.json against its canonical sub-schema:
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

# Inspect a single case via jq:
jq '.cases[0]' vectors/hkdf-sha256/v0.1/cases.json
```

## Regeneration

Each category's vector set is generated by a corresponding `#[ignore]`-gated test in the main repo (`protocol/`), so regenerating vectors is always a deliberate act:

| Category | Generator | Output |
|---|---|---|
| `vault-prefs` | `cargo test -p protocol --lib emit_prefs_vectors_for_paste -- --ignored` | `/tmp/myne_vault_prefs_vectors.txt` (paste-format) |
| `vault_meta_rewrite` | `cargo test -p protocol --lib emit_vector_hex_for_paste -- --ignored` | `/tmp/myne_vault_meta_rewrite_vector.txt` (paste-format) |
| `hkdf-sha256` | `cargo test -p protocol --lib emit_subkey_vectors_for_paste -- --ignored` | `/tmp/myne_hkdf_sha256_vectors.json` (full JSON, copy verbatim) |

`phrase_normalize` vectors are hand-curated against the protocol §7.4 pipeline; no generator.

## Gotchas

- **NFKD normalization order matters** for BIP-39 phrase vectors — apply NFKD before passing to Argon2id, never after. Vectors must include both NFC and NFKD-equivalent inputs to catch implementations that skip the normalization step.
- **Hex encoding is lowercase, no `0x` prefix.** A vector's `expected_<output>_hex` of `1a2b3c...` not `0x1A2B3C...`. Don't mix cases within a single vector file.
- **Vector IDs are immutable** — once a vector is shipped, its `vector_id` is part of the contract; downstream test runners may reference it. Renaming requires a major version bump.
- **Bit-identical means byte-identical** — there is no "approximately equal" for these vectors. Endianness, padding, and reserved bytes all matter. Cross-language tests should diff bytes, not parsed structures.
- **Adding a new category** requires landing a sub-schema file under `schema/v0.1/`, extending the root schema's `schema` enum AND `oneOf` branches in lockstep, and adding the corresponding `cases.json` + integration runner in the main repo.
