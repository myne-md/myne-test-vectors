# myne-test-vectors — JSON test vectors for Myne v0.1 protocol

> Companion repository to [`myne`](https://github.com/myne-md/myne). Public, **CC0** (public domain) — vectors must remain maximally reusable for independent implementations in any language.

## Purpose

Language-agnostic JSON test vectors for the Myne cryptographic protocol. Any independent implementation (Rust, Go, Swift, Kotlin, JavaScript, etc.) MUST produce **bit-identical outputs** from these inputs. A reviewer or contributor verifying a port should not need to read the reference Rust implementation; running their language's test runner against this repository should be enough.

## Status

**Phase 1, generation not yet started.** The `protocol/` Rust crate (in the main repo) is the source of truth from which canonical vectors are generated. Vectors are retroactively populated as Lane A primitives stabilize — see main repo `agent tooling/features/phase-1-plan/omer.md` Lane D items #9–#15 (D1 schema → D7 account-number).

## Vector categories (planned)

Each lives in its own subdirectory under `vectors/` with a small `schema.md` describing the JSON shape and field-by-field meaning:

- `vectors/argon2id/` — KDF outputs at canonical Phase 1 params (`m=256 MiB, t=3, p=2`, Argon2 1.3) + parameter-block AAD binding cases
- `vectors/xchacha20-poly1305-ctx/` — AEAD encrypt/decrypt with CTX commitment outer (Chan & Rogaway 2022) at on-disk tag width of 32 bytes
- `vectors/hkdf-sha256/` — Subkey derivations through canonical helper, exercising every registered role string with `myne.v0.1 ` info-prefix
- `vectors/bip39-recovery/` — 24-word phrase → `recovery_wrap_key` via Argon2id (NOT BIP-39's PBKDF2 step), including NFKD normalization edge cases
- `vectors/account-number/` — Crockford base32 encoding, Damm-32 check digit, liberal-parsing normalization (case, hyphens, `MY-` prefix)
- `vectors/aad-construction/` — Canonical AAD byte assembly for every AEAD call site
- `vectors/vault-meta/` — Full `MYV1` byte strings + decoded fields + tampering rejection cases
- `vectors/myn-blob/` — Full `MY01` byte strings + content-type variations (planned, blocked on main repo `protocol/blob.rs` landing)

## JSON schema convention (D1 — pending)

Every vector file:
- Hex-encoded inputs (`key`, `nonce`, `aad`, `salt`, etc.)
- Hex-encoded expected outputs
- `spec_section` reference (e.g., `"protocol.md §5.1.1"`)
- `vector_id` (stable string, never reused)
- `protocol_version: "v0.1"` — pinned per vector so v005+ vectors can coexist without ambiguity

D1 (master schema definition) lives at `schema/v0.1.json`; CI in this repo validates every vector file against it.

## v005 reservation

Future major-version vectors will live under `vectors/v005-*/` (separate directories from v0.1 vectors). v005 magic bytes: `MY05`/`MYV5`; HKDF prefix: `"myne.v005 "`. v0.1 and v005 are **forward-incompatible by design** — they share this repo for distribution convenience, not for coexistence within a single client.

## Cross-implementation discipline

Independent ports (Phase 2 Go server, Phase 3 web/mobile clients) verify primitive-level compatibility by running these vectors through their language's runner. **A failed vector in any port is a stop-the-world bug** — either the port has drifted or the spec is wrong. Don't regenerate vectors to "fix" a failing port without reading why.

## Common commands

```bash
# Once D1 schema lands:
jq -e '.vector_id' vectors/argon2id/v0.1/wrap-password-short.json
# Validate all vectors against schema:
ajv validate -s schema/v0.1.json -d 'vectors/**/*.json'
```

## Gotchas

- **NFKD normalization order matters** for BIP-39 phrase vectors — apply NFKD before passing to Argon2id, never after. Vectors must include both NFC and NFKD-equivalent inputs to catch implementations that skip the normalization step.
- **Hex encoding is lowercase, no `0x` prefix.** A vector's `expected_output` of `1a2b3c...` not `0x1A2B3C...`. Don't mix cases within a single vector file.
- **Vector IDs are immutable** — once a vector is shipped, its `vector_id` is part of the contract; downstream test runners may reference it. Renaming requires a major version bump.
- **Bit-identical means byte-identical** — there is no "approximately equal" for these vectors. Endianness, padding, and reserved bytes all matter. Cross-language tests should diff bytes, not parsed structures.
