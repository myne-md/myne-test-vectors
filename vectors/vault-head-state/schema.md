# `vault-head-state` vector schema

Pins the byte mapping of `protocol::encode_head_state` + `protocol::decode_head_state` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)). The codec emits and consumes `vault.head-state` — the v0.1 **device-local sync baseline** (`MYH1` magic; MYND-416) that a device writes next to `vault.meta` to remember the last vault head it accepted.

**Reference implementation:** `protocol/src/vault_head.rs::{encode_head_state, decode_head_state}`.
**Spec:** `docs/spec/protocol.md` § 9.16 (the `MYH1` envelope) + § 3.4 registry entry `"myne.v0.1 vault-head-mac"` (the MAC key role) + `docs/spec/sync-protocol.md` § 3 (the wrapped head record itself).

## Relationship to the `vault-head` category

These two categories pin **different layers of the same object** and must be read together:

| | `vault-head` | `vault-head-state` (this file) |
|---|---|---|
| Surface | sync **wire** head | on-disk **baseline** envelope |
| Spec | `sync-protocol.md` § 3–§ 5 | `protocol.md` § 9.16 |
| Magic | none — the wire head is magic-less, versioned by its `head_format_version` byte | `MYH1`, a § 10.3 registered family |
| Pins | `head_mac_input` canonical encoding, `head_mac`, `head_id`, VV dominance | the 6-byte header wrapped around that record |

The `MYH1` header is a **pure prefix**: bytes `[6..]` of a baseline blob are the sealed wire record verbatim. The `encode` cases here use the same fixture bytes as the `vault-head` `encode` cases, so the two files can be lined up field by field — and the runner asserts the prefix property directly rather than trusting it.

## The one non-AEAD blob family

Every other `MY??` family in the § 10.3 table is AEAD-encrypted; `MYH1` is not. The bytes it wraps are exactly the head the server already relays, so there is nothing to hide from an adversary who has the file — integrity is the head's **own** HMAC-SHA-256 under `head_mac_key = DeriveSubkey(root_key, "vault-head-mac")`, re-verified on every load.

Two consequences a porter must not miss:

1. **There is no `decrypt_aead_fail` shape here.** The MAC surface gets its own `mac_fail` case kind.
2. **A MAC failure is deliberately distinguishable.** The § 6.4 non-distinguishing rule exists to stop a decoder from leaking *which* check failed on a confidential blob; `MYH1` carries no confidentiality claim, so there is nothing for a distinguishable error to leak. The comparison itself still has to be constant-time.

## Pipeline (as fixed by the reference implementation)

**Encode side** (`encode_head_state`):

1. **Seal the head** (`VaultHead::seal`) — canonically encode `head_mac_input` (G1 `device_id`-sorted version vector, G2 lexicographically-sorted merge parents), compute `head_mac = HMAC-SHA-256(head_mac_key, head_mac_input)`, and concatenate to form `record = head_mac_input ∥ head_mac`. `head_id = SHA-256(record)`.
2. **Prepend the 6-byte baseline header** per § 9.16:
   - `[0..4]` magic `MYH1`
   - `[4]` protocol_version `0x01`
   - `[5]` head_format_version `0x01` (also carried inside the record, so a tampered header byte cannot be silently accepted)
   - `[6..]` the sealed record verbatim

**Decode side** (`decode_head_state`): length guard → magic → protocol_version → head_format_version → `open_vault_head` (constant-time MAC verification + structural well-formedness). The order is load-bearing: the length guard fires **before** the magic comparison, so a 5-byte input yields `RecordTooShort` rather than a panic on `blob[0..4]`.

## File layout

```
vectors/vault-head-state/
├── schema.md              ← this file
└── v0.1/
    └── cases.json         ← array of 9 KAT cases (encode + decode + MAC-fail + structural parse-fail)
```

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/vault-head-state.json`](../../schema/v0.1/vault-head-state.json) (per-case shape with a 4-way `case_kind` discriminator). Run the repository's documented `ajv-cli validate` command (see the root `README.md`) to verify any `cases.json` against the canonical schema. The sub-schema is authoritative — this document covers only the domain-specific semantics.

Four case kinds carry different field shapes:

- **`encode`** (3 cases: genesis / linear / merge) — pins the encoder's byte output. Inputs are the head's fields plus `input_root_key_hex`; parents and version-vector entries are supplied **unsorted** so the vector also pins the G1/G2 canonical sorts that happen underneath the envelope. Optional `informational_head_mac_key_hex` and `informational_head_id_hex` carry intermediate state for reviewers. Output: `expected_head_state_blob_hex`.
- **`decode_ok`** (1 case) — pins the round-trip, MAC verification included. Recovered `expected_prev_head_hash_hex` / `expected_version_vector` are in **canonical** order, not the caller's input order.
- **`mac_fail`** (1 case) — the final byte of the trailing `head_mac` is flipped. Fixture carries the **real** `root_key`, so a port that never verifies the MAC returns `Ok` and fails loudly here rather than passing on a coincidental error.
- **`parse_fail`** (4 cases) — one per distinguishable pre-MAC structural reject: `state_magic_mismatch`, `protocol_version_mismatch`, `format_unsupported`, `record_too_short`.

## How to consume

A port runs each case based on `case_kind`:

- **`encode`**: build the head from the `input_*` fields, derive `head_mac_key`, seal, wrap with the baseline header, hex-encode, compare byte-equal to `expected_head_state_blob_hex`. Also assert bytes `[6..]` equal the sealed record — the prefix property.
- **`decode_ok`**: decode the blob under `input_root_key_hex` and compare every recovered field.
- **`mac_fail`**: decode and assert the port's equivalent of `MacMismatch`. `Ok` here means the port is not verifying.
- **`parse_fail`**: decode and assert the *specific* structural error named by `expected_error`, before any MAC computation.

Any mismatch is a stop-the-world bug (per repo `README.md` "Cross-implementation discipline").

## Invariants the vectors pin

- **MYH1 magic + version + format bytes.** Encoder output `[0..4]` = `MYH1`, `[4]` = `0x01`, `[5]` = `0x01`. Decoder rejects any other value before MAC verification.
- **The header is a pure prefix.** Bytes `[6..]` are the wire record verbatim — the envelope never re-orders, re-canonicalizes, or re-encodes what it wraps.
- **Reject ordering.** The length guard precedes the magic comparison; magic precedes protocol_version; protocol_version precedes head_format_version; all four precede MAC verification.
- **`head_format_version` forward-migration.** The `parse-fail-format-unsupported` case pins that a v0.1-only client rejects a higher format cleanly, so it re-bootstraps its baseline from the server rather than mis-parsing a newer record.
- **MAC verification is not optional.** The `decode_ok` / `mac_fail` pair is what makes this enforceable: a port can pass the first by parsing fields alone, but only a real verification passes both.
- **G1 / G2 canonical sorts survive the envelope.** The `encode-linear` and `encode-merge` cases supply unsorted input and pin sorted output.

## Regeneration

If the reference implementation changes (must not happen byte-stably without a major-version event), regenerate this vector via:

```bash
cargo test -p protocol --lib emit_head_state_vectors_for_paste -- --ignored
cat /tmp/myne_vault_head_state_vectors.json
# copy the full JSON verbatim into cases.json
```

The regenerator test lives in `protocol/src/vault_head.rs::tests::emit_head_state_vectors_for_paste`. It is `#[ignore]`-gated so it never runs in default `cargo test` invocations, and it **self-asserts at mint**: every failure fixture is fed through `decode_head_state` before it is written out, so a broken generator cannot ship a vector that pins the wrong error.

## What this does NOT cover

- **The wire head itself** — canonical `head_mac_input` encoding, `head_id`, and the § 5.3 VV-dominance decision are the `vault-head` category's job. This one starts from a sealed record and pins only what the baseline envelope adds.
- **The head fingerprint** (`head_fingerprint`, § 3.4 domain `"myne.v0.1 vault-head-fingerprint"`) — pinned by the `head-fingerprint` category.
- **The manifest** whose SHA-256 the head commits — pinned by the `manifest` category. The head commits the hash, never the map.
- **Encode-side structural rejects** (`ParentCountInvalid`, `VersionVectorEmpty`, `VersionVectorNotCanonical`, …) — these fire in `VaultHead::seal` before the envelope exists, so they belong to the wire-head surface rather than the baseline codec.
- **Caller policy on a failed load.** A device that cannot decode its baseline re-bootstraps from the server (`sync-protocol.md` § 6 BootstrapTofu); that is client behavior, not codec behavior.
