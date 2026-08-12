# `vault-meta` vector schema

Pins the byte mapping of `protocol::serialize_vault_meta` + `protocol::parse_vault_meta` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)) — the create-time `MYV1` vault.meta encoder/decoder pair. Distinct from `vectors/vault_meta_rewrite/` which pins only the password-side rewrite path; this directory pins the **full** create-time encode + the **structural-parse-reject** discipline.

**Reference implementation:** `protocol/src/vault_meta.rs::{serialize_vault_meta, parse_vault_meta}`.
**Spec:** `docs/spec/protocol.md` § 9.3 (on-disk 345-byte fixed-offset layout).

## Pipeline (as fixed by the reference implementation)

**Encode side** (`serialize_vault_meta`): pure byte assembly of the `VaultMeta` struct into the 345-byte `MYV1` layout. No CSPRNG, no clock, no AEAD calls — the encoder is structural. Field-by-field offsets:

```
[0..4]    magic `MYV1` (4 B)
[4]       protocol_version 0x01 (1 B)
[5]       reserved 0x00 (1 B)
[6..22]   account_number_hash (16 B)
[22..33]  kdf_params canonical block (11 B — see `types::serialize_kdf_params`)
[33..49]  random_salt_password (16 B)
[49..65]  random_salt_recovery (16 B)
[65..89]  nonce_password_wrap (24 B)
[89..153] enc_root_key_password (64 B = 32 ciphertext + 32 CTX tag)
[153..177] nonce_recovery_wrap (24 B)
[177..241] enc_root_key_recovery (64 B)
[241..257] items_key_id (16 B)
[257..281] items_key_nonce (24 B)
[281..345] enc_items_key (64 B)
```

**Decode side** (`parse_vault_meta`): structural validation only. Rejects via the single non-distinguishing `ProtocolError::InvalidVaultMeta` variant when (a) length is not 345 bytes, (b) magic[0..4] ≠ `MYV1`, (c) protocol_version[4] ≠ 0x01, (d) reserved[5] ≠ 0x00, (e) the 11-byte KDF params block at offset 22..33 fails its own canonical parse rules. **AEAD verification of the wrapped key bytes is NOT performed at this layer** — `enc_root_key_password` + `enc_root_key_recovery` + `enc_items_key` are byte-validated for length only; their CTX tags are checked by downstream `aead_open_ctx` calls when unwrapping. Tampering inside those 64-byte regions therefore survives `parse_vault_meta` and surfaces at unwrap time — out of this category's scope (see the `ctx-aead` vector category for AEAD-stage rejects).

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/vault-meta.json`](../../schema/v0.1/vault-meta.json) (per-case shape with a 2-way `case_kind` discriminator: `roundtrip`, `parse_fail`). Run the repository's documented `ajv-cli validate` command (see the root `README.md`) to verify any `cases.json` against the canonical schema. The sub-schema is authoritative — this document covers only the domain-specific semantics.

Two case kinds carry different field shapes:

- **`roundtrip`** — pins the encoder's byte output and the decoder's parse + re-serialize round-trip property. Inputs: every `VaultMeta` field hex-encoded (`account_number_hash`, `random_salt_password`, `random_salt_recovery`, `nonce_password_wrap`, `enc_root_key_password`, `nonce_recovery_wrap`, `enc_root_key_recovery`, `items_key_id`, `items_key_nonce`, `enc_items_key`) plus the nested `input_kdf_params` object (`m_kib` u32, `t` u32, `p` u8, `argon2_version_hex`). Output: `expected_vault_meta_hex` (the 345-byte serialized blob).
- **`parse_fail`** — pins the structural-rejection surface. Input: `input_blob_hex` (a tampered or malformed blob). Output: `expected_kind: "parse_fail"`. Implementations MUST return their port's equivalent of `Err(ProtocolError::InvalidVaultMeta)` — a single non-distinguishing variant per spec § 6.4. The optional `informational_tamper_description` field documents which bytes were tampered relative to the canonical `roundtrip` case.

## How to consume

A port runs each case based on `case_kind`:

- **`roundtrip`**: decode every `input_*_hex`, parse the `input_kdf_params` object into the port's KDF-params type, build the port's `VaultMeta` equivalent, invoke `serialize_vault_meta`, hex-encode the output, compare byte-equal to `expected_vault_meta_hex`. ADDITIONALLY: invoke `parse_vault_meta` on the expected bytes; the parsed `VaultMeta` must re-serialize to the same expected bytes (round-trip property). Any non-equal result is a stop-the-world bug (per repo `README.md` "Cross-implementation discipline").
- **`parse_fail`**: decode `input_blob_hex`, invoke `parse_vault_meta`, assert the port returns its non-distinguishing parse-rejection variant. Ports MUST NOT distinguish which rule (magic / version / reserved / kdf-params / length) fired — single failure mode at the caller-visible surface per spec § 6.4 closed-enum discipline.

## Invariants the vectors pin

For the 7 cases in `cases.json` (1 `roundtrip` + 6 `parse_fail`):

- **Magic / version / reserved bytes.** Output `[0..4]` = `MYV1`, `[4]` = `0x01`, `[5]` = `0x00`. Decoder rejects any other value at parse time.
- **Field offsets are absolute.** Byte 33 is always the first byte of `random_salt_password`; byte 281 is always the first byte of `enc_items_key`. The 11-byte canonical KDF params block at offset 22..33 is the only variable-content portion of the header.
- **Distinct byte patterns per field.** The canonical fixture uses 0xDA, 0xDB, 0xDC, 0xDD, 0xDE, 0xDF, 0xE0, 0xE1, 0xE2, 0xE3 for the 10 fixed-offset fields (distinct from the `vault_meta_rewrite` fixture's 0xAC/0x11../0x99 patterns) so any swap or off-by-one surfaces as a clear hex mismatch in test assertions. Ports SHOULD use any distinguishable patterns; nothing in the protocol mandates these specific bytes.
- **Non-distinguishing structural-parse-reject surface.** Six tampers — magic byte (byte 0 flipped), reserved byte (byte 5 → 0x01), wrong length (truncated to 344), protocol_version (byte 4 → 0x02), kdf_id (byte 22 → 0x02) and argon2_version (byte 32 → 0x10, the two `parse_kdf_params` rejects, MYND-227) — all collapse to the same `ProtocolError::InvalidVaultMeta` in the Rust reference. The KDF-byte pair pins that a port MUST validate the KDF identity at parse time: accepting an unknown kdf_id/argon2_version yields a vault.meta the client cannot honor (or silently derives a wrong wrap key under). The CTX commitment construction inside the wrapped-key regions makes the AEAD-stage tampering computationally indistinguishable from cross-vault mismatches — but those are verified separately by the `ctx-aead` vector category, not here.

## Regeneration

If the reference implementation changes (must not happen byte-stably without a major-version event), regenerate this vector set via:

```bash
cargo test -p protocol --lib emit_vault_meta_vectors_for_paste -- --ignored
cp /tmp/myne_vault_meta_vectors.json vectors/vault-meta/v0.1/cases.json
```

The regenerator test lives in `protocol/src/vault_meta.rs::tests::emit_vault_meta_vectors_for_paste`. It is `#[ignore]`-gated so it never runs in default `cargo test` invocations. Emits ready-to-copy JSON (matches the `subkey.rs` HKDF generator pattern); the original 3-case matrix is too small to warrant paste-format hex dumps.

The four MYND-227 structural `parse_fail` additions have their own append generator (the original stays byte-stable — vector_id immutability):

```bash
cargo test -p protocol --lib emit_vault_meta_parse_fail_vectors -- --ignored
# splice the JSON array fragment at /tmp/myne_vault_meta_parse_fail_vectors.json into the cases array
```

## The v0.2 side (`v0.2/cases.json`)

`v0.2/` pins `serialize_vault_meta_v0_2` + `parse_vault_meta_v0_2` — the `NTV5`
`vault.meta` of a **migrated** vault (§10.4.1). The layout is this one,
byte-for-byte, 345 bytes, with exactly two substitutions: magic `NTV5` at
`[0..4]` and `protocol_version` `0x02` at `[4]`. Every field after those five
bytes sits at the offset §9.3 already pins.

The two reject case kinds are the load-bearing ones and they are deliberately a
**pair**, because §10.5's anti-downgrade property is bidirectional:

- **`parse_fail`** — the v0.2 parser must refuse an `MYV1` blob.
- **`parse_fail_v0_1`** — the v0.1 parser must refuse an `NTV5` blob. This is the
  direction that matters for a mixed fleet: an un-migrated client meeting a
  migrated vault has to refuse it cleanly rather than half-parse a structure
  whose meaning it does not know.

A `roundtrip` case alone would pass in an implementation that ignored the magic
entirely, which is precisely the implementation the pair exists to fail.

Schema: [`schema/v0.2.json`](../../schema/v0.2.json) +
[`schema/v0.2/vault-meta.json`](../../schema/v0.2/vault-meta.json).
Generator: `emit_vault_meta_v0_2_vectors_for_paste` (`protocol/src/vault_meta_v0_2.rs`).
Runner: `vault_meta_v0_2_vectors` (`protocol/tests/vectors.rs`).

## What this does NOT cover

- **Password-side rewrite path.** Pinned by `vectors/vault_meta_rewrite/` — that category covers the `rewrite_vault_meta_password_side` function which preserves the recovery + items_key fields byte-for-byte while re-deriving the password-side under fresh KDF inputs. This category covers the full create-time encoder.
- **AEAD-stage tampering reject cases.** The 64-byte `enc_root_key_*` and `enc_items_key` regions carry CTX-committed ciphertext + tag pairs. Tampering inside those regions surfaces at `aead_open_ctx` time (downstream of `parse_vault_meta`), not at parse time. Covered by the `ctx-aead` vector category.
- **KDF params range checks.** The kdf_id + argon2_version byte rejects ARE pinned here (MYND-227); range checks on `m_kib`/`t`/`p` are deliberately deferred to the Argon2id call site in the reference implementation (a parse-layer range gate would hard-code runtime affordability into the format) and are not vectorable as parse rejects.
- **Oversize blob lengths.** The wrong-length case pins truncation (344 bytes); a 346-byte probe would exercise the same `len != 345` check from the other side and is omitted as redundant.
