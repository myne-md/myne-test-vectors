# `vault-meta-ntv8` — v0.2 password-only `vault.meta`

Cross-implementation vectors for `protocol::vault_meta_ntv8`
(`serialize_vault_meta_ntv8`, `parse_vault_meta_ntv8`,
`rewrite_vault_meta_password_side_ntv8`); myne-md/myne#938.

`NTV6` made the vault B2B: an owner-wrapped group key, a group-key epoch and a
signed roster. `NTV7` kept that tail **byte for byte** and added a second
credential, a person envelope sealing `root_key` to the owner's identity key.
`NTV8` removes that second credential. The master password is the only vault
credential; the identity phrase is an identity, and it opens nothing here.

## Frame

```
0    4    magic "NTV8"
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
```

Total length is `445 + roster_len`, trailing bytes invalid. There is no person
wrap and no placeholder for one.

## The absence is the length, not a zeroed block

An `NTV7` frame is the same bytes plus a fixed 1208-byte envelope after the
roster, so the two generations differ in exactly two things: the magic, and
1208 bytes of length. That makes `parse-fail-person-wrap-present` the
load-bearing rejection in this file. A reader that keys on the tail layout alone
accepts an `NTV7` file, parses every field correctly, and then waits for a
credential the user no longer has. `parse-fail-ntv6-magic` and
`parse-fail-ntv7-magic` are the same rule from the other side: the tail is
byte-identical across all three generations, so nothing but the magic can
separate them.

## The all-zero rule is structural, as it was on `NTV7`

A frame carrying any byte in the recovery salt, nonce or wrapped key is
**invalid**, not "a vault whose saved phrase may or may not still work". The
writer refuses to emit one and the parser refuses to accept one, and each of the
three fields has its own `parse_fail` case so the rule is checked per field
rather than once.

The fields are kept rather than compacted away, and that is deliberate: their
offsets belong to the §9.3 base layout every generation shares, so removing them
would move `items_key_id` and the `itemsKey` wrap. That turns a truncation of one
frame into a re-wrap of every vault.

## Case kinds

- `roundtrip` (1): parse then re-serialize is byte-identical, and the tail
  fields plus the base fields are read back at their offsets. The frame's own
  Argon2id parameters are pinned too, because a port that ignores them and uses
  its own defaults derives a different wrap key while parsing every byte
  correctly.
- `password_opens` (1): the frame, opened with its master password alone.
  Argon2id over the `"myne.v0.2 argon2id-password|"` salt domain with the
  frame's own KDF parameters and `account_number_hash`, then AEAD-open of
  `enc_root_key_password` under the `password-wrap` role bound to those
  parameters. **This is the case that fails when a port's salt domain, AAD role
  or 11-byte KDF block drifts**, which is the failure no parse-side case can
  see.
- `password_open_fail` (1): a wrong password. One non-distinguishing AEAD
  failure (§6.4); a port must not distinguish it from a tampered frame.
- `upgrade` (1): an `NTV7` frame and the `NTV8` frame it must become on the first
  unlock. Every byte from offset 4 to the end of the roster is identical, so the
  step truncates and **re-wraps no key** — which is why the same password opens
  both frames to the same `root_key`, carried in the same case. A port that
  re-derives or re-wraps anything produces different bytes, and one that re-wraps
  the root key produces a frame this password no longer opens.
- `parse_fail` (10): bad magic (twice, `NTV6` and `NTV7`), bad version, bad
  reserved byte, each of the three phrase-wrap fields, a trailing byte, epoch 0,
  and the whole `NTV7` frame rejected on length. All collapse to ONE
  non-distinguishing result, so `expected_kind` is always `invalid_meta`.

## What crosses the boundary, and what does not

Both sets carry the expected `vault.meta` bytes **and** the `root_key` those
bytes open to under a fixed password. That pairing is the point: a mismatch then
names which half diverged, rather than leaving a port author to guess whether the
frame, the salt domain or the KDF block is wrong.

The committed files are reproducible because every nonce, salt and key comes from
the fixture. Nothing here depends on the seal's randomness — this generation has
none.

## File layout

```
vectors/vault-meta-ntv8/
├── schema.md              ← this file
└── v0.2/
    └── cases.json         ← 14 cases
```

## Regeneration

```bash
cd ../myne && cargo test -p protocol --lib emit_vault_meta_ntv8_vectors_for_paste -- --ignored
# then paste /tmp/myne_vault_meta_ntv8_vectors.json over this directory's cases.json
```

The emitter asserts every case before emitting (each `parse_fail` blob is
rejected, the fixture password opens the frame and the wrong one does not, and the
upgrade's surviving bytes are identical), so a broken fixture fails generation
rather than being frozen.
