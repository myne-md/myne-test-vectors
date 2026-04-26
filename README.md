# myne-test-vectors

Language-agnostic **JSON test vectors** for the [Myne](https://github.com/myne-md/myne) cryptographic protocol.

For the protocol specification, see [`myne/spec/protocol.md`](https://github.com/myne-md/myne/blob/main/spec/protocol.md).

## Status

**Phase 1, not yet started.** This repository will hold canonical test vectors covering KDF (Argon2id), AEAD (XChaCha20-Poly1305 + CTX commitment), HKDF subkey derivation, BIP-39 recovery derivation, and account-number generation/validation. Vectors are language-agnostic so any Myne implementation in any language can verify against the same canonical references.

## License

CC0 1.0 — public domain. Test vectors should be maximally reusable across implementations.
