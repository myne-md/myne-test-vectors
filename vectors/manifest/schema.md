# `manifest` vectors (v0.1)

Byte-level KAT for the **vault-head manifest** — the canonical
`uuid → content_nonce` map whose SHA-256 the vault head commits as
`manifest_hash`. Spec `docs/spec/sync-protocol.md` **§4.2** (MYND-414).
Reference implementation: `protocol/src/manifest.rs`.

The head keeps only `manifest_hash` (§3.2), so the head stays O(1) in vault size.
This category pins the **canonical byte encoding of the manifest object itself** —
the input whose hash the head anchors, and which the Rust client and the Go
server-adjacent tooling must produce bit-identically.

## What the manifest covers (§4.2)

- **Content blobs only:** notes (`.myn`, keyed by note UUID) and attachments
  (`MYA1`, keyed by `attachment-id`). Both keys are 16 bytes and share one flat
  keyspace — the manifest is uniform on a single discriminator.
- **Not** the synced singletons (`vault.prefs` / `vault.meta` /
  `vault.dictionary` / `vault.attachments`), whose rollback is the accepted
  "local prefs replay" limitation (`threat-model.md` §7).
- **Discriminator = `content_nonce`** (24 B, `protocol.md` §9.2): freshly random
  per content save and readable from the cleartext header **without decrypting**,
  which is what gives §4.2 its two-layer (pre-decrypt + on-decrypt) rollback
  detection. A `deleted`-marked tombstone version carries its own fresh
  `content_nonce` and stays in the manifest until horizon-GC.

## Canonical encoding (encode-then-hash, §3 discipline)

```
manifest_canonical =
    "myne.v0.1|vault-manifest|"   (25 B  domain prefix)
 || protocol_version              ( 1 B  = 0x01)
 || manifest_format_version       ( 1 B  = 0x01; forward hook)
 || entry_count                   ( 4 B  u32 big-endian)
 || for each entry, SORTED ASCENDING BY id (16 B byte-order), id unique:
        id                        (16 B  note UUID or attachment-id)
        content_nonce             (24 B  the per-revision discriminator)

manifest_hash = SHA-256(manifest_canonical)                        (32 B)
```

Fixed-width, count-prefixed, deterministic entry order — the same discipline as
the head MAC input (§3.2) and the CTX commitment (`protocol.md` §5.1.3).

**The domain prefix is load-bearing.** `"myne.v0.1|vault-manifest|"` is 25 bytes,
distinct from the head record's `"myne.v0.1|"` + version bytes, so a manifest can
never be reinterpreted as a head record (§3.5 hash domain separation). The
`domain_prefix_mismatch` parse case pins that a decoder rejects rather than
best-effort-parses.

## Cases (7)

- **`encode`** — pin `expected_manifest_bytes_hex` (the full canonical encoding)
  and `expected_manifest_hash_hex` (its SHA-256) for a set of `input_entries`.
  Entries are supplied **unsorted** where relevant, so the vectors pin the
  ascending-by-id sort rather than trusting the caller. The empty manifest (a
  fresh vault) is a valid case: `entry_count = 0`, header bytes only.
- **`parse_fail`** — pin a distinguishable structural reject via
  `expected_error`: `domain_prefix_mismatch`, `format_unsupported`,
  `trailing_bytes`. `format_unsupported` is the forward hook — a future
  `manifest_format_version` must be rejected cleanly rather than mis-parsed;
  `trailing_bytes` pins the **strict** parse (a manifest with extra bytes after
  the declared entries is malformed, not tolerated).

## How to consume

- **`encode`**: build the manifest from `input_entries`, canonically encode,
  compare `expected_manifest_bytes_hex` byte-for-byte, then hash and compare
  `expected_manifest_hash_hex`. Pinning both means a port whose hashing is right
  but whose encoding drifts still fails — and it fails at the layer where the
  bug is.
- **`parse_fail`**: decode `input_manifest_bytes_hex` and assert the *specific*
  error named by `expected_error`.

Any mismatch is a stop-the-world bug (per repo `README.md` "Cross-implementation
discipline"). These vectors are the Rust ↔ Go bit-identity contract.

## What this does NOT cover

- **The head that commits the hash** — [`vault-head`](../vault-head/schema.md).
- **How a device reconciles a manifest mismatch** (re-fetch, fork audit) — client
  policy in `sync-protocol.md` §5–§6, not codec behavior.
- **The content blobs themselves** — [`myn-blob`](../myn-blob/schema.md) and
  [`attachment-blob`](../attachment-blob/schema.md). The manifest sees only ids
  and nonces, never plaintext or ciphertext.

## Regeneration

```
cargo test -p protocol --lib emit_manifest_vectors_for_paste -- --ignored
cp /tmp/myne_manifest_vectors.json vectors/manifest/v0.1/cases.json
```

Generator: `emit_manifest_vectors_for_paste` in `protocol/src/manifest.rs`.
Runner: `manifest_vectors` in `protocol/tests/vectors.rs` (skip-if-missing).
