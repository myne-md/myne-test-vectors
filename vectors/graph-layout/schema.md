# `graph-layout` vectors (v0.1)

Byte-level KAT for the **`MYG1` `vault.graph`** encrypted knowledge-graph
layout-cache codec — `protocol/src/graph_layout_blob.rs`
(`encode_graph_layout_blob_raw` / `decode_graph_layout_blob`), spec
`docs/spec/protocol.md` **§9.19** (+ §3.4 subkey, §5.2 AAD, §6.4 boundary).

A structural twin of [`vault-workspace`](../vault-workspace/schema.md) (`MYW1`):
same 34-byte fixed-offset envelope, same **opaque** inner plaintext (the desktop
client's serialized node-position map — no canonical-JSON discipline), same
four case kinds. It differs only in identity: magic `MYG1`, KEK role
`graph-layout-key`, AAD role `graph-layout-blob`, 16 MiB plaintext cap. A
**rebuildable cache** — never synced, never backed up; a rejected blob falls to
recomputing the layout, never data loss.

## On-disk layout (§9.19.1)

```
Offset  Size  Field                        Notes
0       4     magic                        ASCII "MYG1"
4       1     protocol_version             0x01
5       1     graph_layout_format_version  0x01 (initial position-map schema)
6       24    nonce                        XChaCha20-Poly1305 nonce
30      4     ciphertext_len               big-endian uint32, plaintext_len + 32
34      M     ciphertext                   AEAD ciphertext + 32-byte CTX tag
```

## Key + AAD lineage

- `graph_layout_key = DeriveSubkey(root_key, SubkeyRole::GraphLayoutKey)` — HKDF
  context `"myne.v0.1 graph-layout-key"` (§3.4). `root_key`-derived, itemsKey-
  independent (like prefs/search/indices/workspace).
- AAD `aad_graph_layout_blob(account_number_hash)` — prefix `"myne.v0.1|"` ∥
  `0x01` ∥ u16_be(17) ∥ `"graph-layout-blob"` ∥ `account_number_hash(16)`, total
  **46 bytes**. The HKDF role `graph-layout-key` is **DISTINCT** from the AAD
  role `graph-layout-blob`.

## Cases (12)

- **encrypt** ×3 — empty / small position-map (`{"v":1,"pos":[[id,x,y],...]}`) /
  1-KB opaque. Pin the `MYG1` byte output. The zero-plaintext case is the
  smallest valid blob (66 bytes). `input_root_key_hex` exercises the full
  root_key → KEK → blob lineage.
- **decrypt_ok** ×1 — small position-map round-trip.
- **decrypt_aead_fail** ×2 — (a) CTX-tag tamper, (b) cross-vault (wrong
  `account_number_hash` via the AAD bind). Both collapse to
  `GraphLayoutBlobError::CannotDecrypt`, non-distinguishing per §6.4.
- **parse_fail** ×6 — one per distinguishable pre-decryption reject
  (`magic_mismatch`, `protocol_version_mismatch`, `format_unsupported`,
  `header_too_short`, `ciphertext_truncated`, `ciphertext_len_overflow`). These
  run before decryption; the caller collapses them to "recompute the layout"
  (§9.19.4), but a port MUST surface the specific reject.

`PlaintextTooLarge` + `RandomnessUnavailable` are encode-side and cannot be
expressed as a decode-a-blob case, so they are not vectored.

## Regeneration

```
cargo test -p protocol --lib emit_graph_layout_vectors_for_paste -- --ignored
cp /tmp/myne_graph_layout_vectors.json vectors/graph-layout/v0.1/cases.json
```

Generator: `emit_graph_layout_vectors_for_paste` in
`protocol/src/graph_layout_blob.rs`. Runner: `graph_layout_vectors` in
`protocol/tests/vectors.rs` (skip-if-missing).
