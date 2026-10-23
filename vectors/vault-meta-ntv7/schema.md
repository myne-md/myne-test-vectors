# `vault-meta-ntv7` — v0.2 person-wrapped `vault.meta`

Cross-implementation vectors for `protocol::vault_meta_ntv7`
(`serialize_vault_meta_ntv7`, `parse_vault_meta_ntv7`,
`seal_root_key_to_person_raw`, `open_person_wrapped_root_key`); myne-md/myne#911.

`NTV6` made the vault B2B: an owner-wrapped group key, a group-key epoch and a
signed roster. `NTV7` keeps that tail **byte for byte** and changes the
credential side: the password wrap stays, and the recovery-phrase wrap is
**retired** in favour of the person's identity phrase sealing `root_key` to the
owner's person key.

## Frame

```
0    4    magic "NTV7"
4    1    protocol_version 0x02
5    1    reserved 0x00
6    339  the base layout of protocol.md §9.3, with the three phrase-wrap
          fields zero (recovery salt at 49, recovery nonce at 153,
          recovery-wrapped root_key at 177)
345  24   group_key_nonce
369  64   enc_group_key
433  8    group_key_epoch            (big-endian, never 0)
441  4    signed_roster_len          (big-endian)
445  n    signed_roster_envelope
+n   1088 person wrap: ML-KEM-768 ciphertext
+1088 32  person wrap: classical ephemeral Ed25519 public key
+1120 24  person wrap: AEAD nonce
+1144 64  person wrap: root_key ciphertext (32) || CTX commitment (32)
```

Total length is `445 + roster_len + 1208`, trailing bytes invalid.

**The wrapped root key is 64 bytes, not 80.** `aead_seal_ctx` emits
`ciphertext || T'` and the Poly1305 tag is recomputed at open rather than
stored, which is also why `enc_root_key_password` is 64 bytes. An 80-byte reading
(ciphertext + stored tag + commitment) double-counts the tag; the sub-schema's
`{128}` hex pattern is what makes that arithmetic error fail rather than ship.

## The all-zero rule is structural

A frame carrying any byte in the recovery salt, nonce or wrapped key is
**invalid**, not "a vault whose phrase may or may not still work". The writer
refuses to emit one and the parser refuses to accept one, and each of the three
fields has its own `parse_fail` case so the rule is checked per field rather
than once.

## The person wrap's AAD

Role `"person-wrap-root-key"` (v0.2 AAD prefix + version + length-prefixed role),
then the recipient fingerprint (32) and the frame's own `items_key_id` (16): 80
bytes. Deliberately **not** the vault id (a joining device mints its own) and
**not** `account_number_hash` (this generation keeps the account number out of
vault cryptography). `person-wrap-fail-wrong-vault` is the case that pins why
`items_key_id` is there at all.

## Case kinds

- `roundtrip` (1): parse then re-serialize is byte-identical, and the tail
  fields plus the five person-wrap slices are read back at their offsets. The
  re-serialize half is what catches a port that reads every field and writes a
  different frame.
- `parse_fail` (8): bad magic, bad version, bad reserved byte, each of the three
  phrase-wrap fields, a trailing byte, and epoch 0. All collapse to ONE
  non-distinguishing result, so `expected_kind` is always `invalid_meta`.
- `person_wrap_open` (1): the owner's keypair, derived from three seeds, opens
  the envelope to the vault's `root_key`.
- `person_wrap_open_fail` (3): tampered ML-KEM ciphertext, tampered nonce, and
  the same frame opened with another vault's `items_key_id`.

**The seal's randomness is never pinned.** ML-KEM encapsulation is randomized, so
no port can be asked to mint our exact envelope. What crosses the boundary is
deterministic keys (from the seeds) and the OUTCOME (this root key, or a
refusal). The committed file is reproducible for us because the generator drives
the seal from a fixed seed; that is an emitter property, not part of the
contract.

## File layout

```
vectors/vault-meta-ntv7/
├── schema.md              ← this file
└── v0.2/
    └── cases.json         ← 13 cases
```

## Regeneration

```bash
cd ../myne && cargo test -p protocol --lib emit_vault_meta_ntv7_vectors_for_paste -- --ignored
# then paste /tmp/myne_vault_meta_ntv7_vectors.json over this directory's cases.json
```

The emitter asserts every case before emitting (each `parse_fail` blob is
rejected, each `open_fail` refuses, the good one opens), so a broken fixture
fails generation rather than being frozen.
