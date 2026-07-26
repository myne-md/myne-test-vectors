# `vault-head` vectors (v0.1)

Byte-level KAT for the **Phase 2 sync vault head** — the fork-consistency
primitive of `docs/spec/sync-protocol.md` **§3–§5** (+ `protocol.md` §3.4 for the
MAC key role). Reference implementation: `protocol/src/vault_head.rs`
(`VaultHead::seal`, `compute_vault_head_mac`, `compute_head_id`,
`open_vault_head`, `vv_dominance`).

The head is a small, server-relayed, per-push record that lets one user's
devices detect a malicious server trying to **roll back**, **withhold**, or
**fork** their vault — without the server ever being able to read or forge
content. It is a **sync-wire object with no on-disk magic**: versioned by its own
`head_format_version` byte, not by a §10.3 blob family. The device-local
envelope that wraps it for storage (`MYH1` `vault.head-state`, `protocol.md`
§9.16) is a separate surface, pinned by
[`vault-head-state`](../vault-head-state/schema.md).

**HMAC, not a signature** (§2): single-user, so every device shares `root_key`
and a symmetric MAC suffices — no new primitive enters the closed RustCrypto set.

## Canonical record (§3.2–§3.4)

```
head_mac_input =
    "myne.v0.1|"              (10 B  domain prefix)
 || protocol_version          ( 1 B  = 0x01)
 || head_format_version       ( 1 B  = 0x01; §3.6 forward hook)
 || account_number_hash       (16 B  vault binding)
 || manifest_hash             (32 B  SHA-256 of the canonical manifest, §4.2)
 || seq                       ( 8 B  u64 BE; auxiliary Lamport scalar)
 || parent_count              ( 1 B  0 = genesis / 1 = linear / 2 = merge)
 || prev_head_hash[]          (32 B each, BYTE-SORTED ascending — gap G2)
 || vv_len                    ( 4 B  u32 BE)
 || version_vector            (entry_count u32 BE, then per entry:
                               device_id (16 B) || counter (8 B BE),
                               SORTED by device_id, unique — gap G1)

head_mac = HMAC-SHA-256(head_mac_key, head_mac_input)          (32 B, untruncated)
record   = head_mac_input || head_mac
head_id  = SHA-256(record)                                     (32 B, gap G3)
```

`head_mac_key = DeriveSubkey(root_key, SubkeyRole::VaultHeadMac)` — HKDF context
`"myne.v0.1 vault-head-mac"` (§3.4), pinned independently in the `hkdf-sha256`
category. The HMAC is realized via `Hkdf::<Sha256>::extract` (RFC 5869 §2.2), the
same construction as `compute_attachment_id` — no `hmac` crate, no new dependency.

**`head_id` binds the MAC, not just the content** (gap G3): a parent reference is
a hash over `head_mac_input ∥ head_mac`, so re-MACing the same content under a
different key yields a different chain position.

**`author_device_id` is implicit** (§4.1 reconciliation): the byte layout carries
no explicit author field. The authoring device is the version-vector component
equal to `max(parents) + 1`, derived during verification when the parents are on
hand (§5.4 lazy merge-verify). The byte layout these vectors pin is authoritative
over §4.1's "explicit author for simplicity" wording.

## Cases (8)

- **`encode`** ×4 — pin `expected_head_mac_input_hex`, `expected_head_mac_hex`,
  and `expected_head_id_hex` for: genesis (`parent_count=0`, VV `{A:1}`), a
  linear successor, a **merge** whose parents and VV entries are supplied
  **UNSORTED** (so the vector pins the G2 byte-sort and the G1 `device_id` sort
  rather than trusting the caller), and a tombstone-bearing manifest. Optional
  `informational_head_mac_key_hex` carries the derived MAC key so a reviewer can
  inspect the intermediate state without re-deriving.
- **`verify`** ×4 — pin the §5.3 on-pull VV-dominance decision as
  `expected_decision`: `extension` (candidate ≥ baseline componentwise, ≥1
  strictly greater → accept), `rollback` (some component below baseline →
  reject), `incomparable_merge` (some greater, some less → concurrent, author a
  merge head), and `bootstrap_tofu` (empty baseline → §6 trust-on-first-use).
  Their genesis → linear → merge VV progression is the CAS version transition a
  port must reproduce. Fields are just the two version vectors
  (`input_baseline_version_vector` / `input_candidate_version_vector`) — no keys
  are involved; dominance is a pure comparison.

`encode` cases also carry `input_author_device_id_hex`. It documents *which*
device authored the head, but it is **not** a field of `head_mac_input` — a port
that appends it to the record produces different bytes and fails. It is the VV
component that will equal `max(parents) + 1`, i.e. the implicit-author property
above stated as data.

## How to consume

- **`encode`**: build the head from the `input_*` fields **in the given order**,
  derive `head_mac_key`, and compare all three expected values byte-for-byte. A
  port that canonicalizes correctly produces identical bytes even though the
  merge case's inputs arrive unsorted.
- **`verify`**: run the port's dominance comparison over
  (`input_baseline_version_vector`, `input_candidate_version_vector`) and compare
  the decision string. Note `bootstrap_tofu` is *not* a dominance result — it is
  the empty-baseline case, and a port that folds it into `extension` accepts a
  first-contact head it should have TOFU-pinned.

Any mismatch is a stop-the-world bug (per repo `README.md` "Cross-implementation
discipline"). These vectors are the Rust ↔ Go bit-identity contract: the same
file drives `myne-server`'s Go implementation.

## What this does NOT cover

- **The `MYH1` on-disk envelope** — [`vault-head-state`](../vault-head-state/schema.md).
- **The manifest** whose SHA-256 the head commits — [`manifest`](../manifest/schema.md).
  The head commits the hash; it never sees the map.
- **The SAS fingerprint** derived from `head_id` — [`head-fingerprint`](../head-fingerprint/schema.md).
- **Structural encode rejects** (`ParentCountInvalid`, `PrevHeadHashNotSorted`,
  `VersionVectorNotCanonical`, `VersionVectorEmpty`) and parse rejects
  (`RecordTooShort`, `TrailingBytes`, `DomainPrefixMismatch`, …) — exercised by
  in-crate unit tests in `vault_head.rs`, not yet vectored.

## Regeneration

```
cargo test -p protocol --lib emit_vault_head_vectors_for_paste -- --ignored
cp /tmp/myne_vault_head_vectors.json vectors/vault-head/v0.1/cases.json
```

Generator: `emit_vault_head_vectors_for_paste` in `protocol/src/vault_head.rs`
(MYND-418, self-asserting at mint). Runner: `vault_head_vectors` in
`protocol/tests/vectors.rs` (skip-if-missing).
