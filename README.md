# myne-test-vectors

Language-agnostic **JSON test vectors** for the [Myne](https://github.com/myne-md/myne) cryptographic protocol.

For the protocol specification, see [`docs/spec/protocol.md`](https://github.com/myne-md/myne/blob/main/docs/spec/protocol.md) in the main repository.

## Status

**Phase 1, generation not yet started.** The Phase 1 desktop scaffold has landed in the main repo, but the Rust crypto protocol crate has not been implemented yet — and that crate will be the source of truth from which canonical vectors are generated. Until then, this repository is a placeholder.

## Planned vector categories

Each category will live in its own subdirectory under `vectors/` with a small `schema.md` describing the JSON shape and the field-by-field meaning. Every vector pins the protocol version (`v0.1` for the initial set) so that future major-version vectors (v005 and beyond) can coexist without ambiguity.

- **`vectors/argon2id/`** — KDF outputs for fixed inputs at the canonical Phase 1 parameters (`m=256 MiB, t=3, p=2`, Argon2 1.3) plus parameter-block AAD binding cases.
- **`vectors/xchacha20-poly1305-ctx/`** — AEAD encrypt/decrypt cases including the CTX commitment outer hash (Chan & Rogaway 2022) at the on-disk tag width of 32 bytes.
- **`vectors/hkdf-sha256/`** — Subkey derivations through the canonical helper, exercising the `myne.v0.1 ` info-prefix discipline and every registered role string.
- **`vectors/bip39-recovery/`** — 24-word phrase → `recovery_wrap_key` derivation through Argon2id (NOT through BIP-39's PBKDF2 step), including NFKD normalization edge cases.
- **`vectors/account-number/`** — Crockford base32 encoding, Damm-32 check digit generation and validation, liberal-parsing normalization (case, hyphens, the `MY-` prefix).
- **`vectors/aad-construction/`** — Canonical AAD byte assembly for every AEAD call site, ensuring no implementation can silently disagree on the bound metadata.

## Why language-agnostic JSON

Any independent implementation of the Myne protocol — in Rust, Go, Swift, Kotlin, JavaScript, or anything else — must produce bit-identical outputs from these inputs. A reviewer or contributor verifying a port should not need to read the reference Rust implementation; running the test runner of their language against this repository should be enough.

## License

[CC0 1.0](LICENSE) — public domain. Test vectors are reference material for independent implementations; they should be maximally reusable.
