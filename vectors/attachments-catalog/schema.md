# `attachments-catalog` vector schema

Pins the byte mapping of `protocol::encode_attachments_catalog_raw` + `protocol::decode_attachments_catalog` + `protocol::aad_attachments_catalog` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)). The codec emits and consumes `vault.attachments` — the v0.1 vault-level singleton encrypted **attachments catalog** (`MYC1` magic, MYND-164) that lives next to `vault.meta` / `vault.prefs` / `vault.search` and maps attachment-ids to their display metadata.

**Reference implementation:** `protocol/src/attachments_catalog.rs::{encode_attachments_catalog_raw, decode_attachments_catalog, aad_attachments_catalog, read_or_empty}`.
**Spec:** `docs/spec/protocol.md` § 9.10 (`MYC1` envelope + canonical-JSON inner shape + forward-compat + `read_or_empty` collapse) + § 5.2 (AAD formula, role `"attachments-catalog"`) + § 3.4 registry entry `"attachments-catalog-key"` (the KEK role).

## Canonical-JSON inner plaintext (like `vault-prefs`, unlike `vault-search`)

The inner plaintext is **canonical JSON**: an object mapping lowercase-hex `attachment_id` → `{content_type, created_at, filename, size}`. Canonical form fixed by the reference implementation:

- Top-level keys (attachment-ids) in lexicographic order; per-entry keys in the fixed order `content_type` < `created_at` < `filename` < `size`.
- No insignificant whitespace; string fields NFC-normalized; empty catalog = `{}` (2 bytes, `7b7d`).
- **Forward-compat (§ 9.10.2):** a decoder MUST tolerate unknown additive per-entry fields (skip them, recover the known fields) — pinned by the `decrypt-unknown-field-tolerated` case.

The catalog is **reconstructible metadata** (display names for the library view): under `read_or_empty` semantics (§ 9.10.4) a missing or undecryptable `vault.attachments` collapses to the empty catalog rather than failing the unlock — lower-severity than note blobs, but the byte mapping is still a cross-implementation contract.

## Two distinct load-bearing role strings

- **KEK (HKDF) role** `"attachments-catalog-key"` (23 ASCII bytes) — `catalog_key = DeriveSubkey(root_key, "attachments-catalog-key")` per § 3.2 + § 3.4. Pinned independently in the `hkdf-sha256` category. Domain-separation pair with `"attachments-id-hmac"` (§ 9.10.1; see the `attachment-blob` category).
- **AAD role** `"attachments-catalog"` (19 ASCII bytes, length-prefixed) — the role inside `aad_attachments_catalog`: `"myne.v0.1|"` (10 B) + protocol_version `0x01` (1 B) + 2-B BE length `0x0013` + role (19 B) + `account_number_hash` (16 B) = 48 bytes. Pinned by the `aad-construction` case.

## Pipeline (as fixed by the reference implementation)

**Encrypt side** (`encode_attachments_catalog_raw`):

1. **Derive `catalog_key`** via the canonical helper: `derive_subkey(root_key, SubkeyRole::AttachmentsCatalogKey)`.
2. **Canonical-JSON-encode** the catalog (rules above; 16-MiB plaintext cap).
3. **AEAD-seal** under `catalog_key` + `catalog_nonce` + the 48-byte AAD (XChaCha20-Poly1305 + 32-byte CTX commitment per § 5.1).
4. **Assemble the outer envelope** (total `34 + ciphertext_len` bytes):
   - `[0..4]` magic `MYC1`
   - `[4]` protocol_version `0x01`
   - `[5]` reserved `0x00`
   - `[6..30]` `catalog_nonce` (24 B)
   - `[30..34]` `ciphertext_len` (big-endian uint32, `plaintext.len() + 32`)
   - `[34..]` ciphertext + 32-byte CTX commitment tag

**Decrypt side** (`decode_attachments_catalog`): parse + validate the header before any decryption, rebuild the AAD from `account_number_hash`, AEAD-decrypt, parse the canonical JSON tolerantly (unknown additive per-entry fields ignored). AEAD failure collapses to the single non-distinguishing `CannotDecrypt` per § 6.4; `read_or_empty` then falls back to `{}`.

Encode is pure and deterministic given the caller-supplied nonce (`_raw` convention); production callers use the `encode_attachments_catalog` wrapper that pulls the nonce from `getrandom`.

## File layout

```
vectors/attachments-catalog/
├── schema.md              ← this file
└── v0.1/
    └── cases.json         ← array of 7 KAT cases (encrypt + decrypt + AEAD-fail + AAD + forward-compat tolerance)
```

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/attachments-catalog.json`](../../schema/v0.1/attachments-catalog.json) (per-case shape with a 5-way `case_kind` discriminator). The sub-schema is authoritative — this document covers only the domain-specific semantics.

- **`encrypt`** (`encrypt-empty`, `encrypt-populated`) — pins the encoder's byte output over the canonical JSON, including the empty `{}` catalog. Inputs: `input_root_key_hex` (the runner derives `catalog_key`), `input_account_number_hash_hex`, `input_plaintext_canonical_utf8_hex`, `input_catalog_nonce_hex`; optional `informational_catalog_key_hex`. Output: `expected_catalog_blob_hex`.
- **`decrypt_ok`** (`decrypt-ok`, `decrypt-unknown-field-tolerated`) — pins the decoder's success path. `expected_plaintext_canonical_utf8_hex` is the **canonical re-encode of the decoded catalog**: for ordinary round-trips it equals the sealed plaintext; for the tolerance case the sealed plaintext (carried verbatim in `informational_sealed_plaintext_utf8_hex`) contains an unknown additive field (`future_flag`, in sorted position between `filename` and `size`) that the decoder MUST ignore — comparison is structural.
- **`decrypt_aead_fail`** (`decrypt-fail-tamper`, `decrypt-fail-cross-vault`) — non-distinguishing failure surface (`expected_kind: "aead_fail"`).
- **`aad_construction`** (`aad-construction`) — the 48-byte `aad_attachments_catalog` canonical bytes.
- **`parse_fail`** (`parse-fail-*`, MYND-227) — the **distinguishable** reject surface, one case per `CatalogBlobError` variant reachable from a decode: the six pre-AEAD structural rejects (`header_too_short`, `magic_mismatch`, `protocol_version_mismatch`, `reserved_byte_nonzero`, `ciphertext_len_overflow`, `ciphertext_truncated`) plus the post-AEAD `plaintext_parse_failed` (a validly-sealed blob whose inner plaintext is not a canonical JSON object). `expected_error` names the variant. The cases carry the REAL `root_key` deliberately: header bytes are not AAD-bound, so a port that skips a structural check decrypts the tampered blob successfully and returns Ok — which the runner catches. Callers then fall back to the empty catalog per § 9.10.4 — the distinguishability is an observability contract.

## How to consume

A port runs each case based on `case_kind`: derive the KEK, invoke the equivalent encoder/decoder/AAD-builder, compare byte-equal — except `decrypt_ok`, where the recovered catalog is compared **structurally** (decode both the blob and `expected_plaintext_canonical_utf8_hex`, compare the parsed catalogs) so forward-compat tolerance cases hold. Any mismatch is a stop-the-world bug (per repo `README.md` "Cross-implementation discipline").

## Invariants the vectors pin

- **MYC1 magic + version + reserved bytes** and the fixed-offset header layout above.
- **Canonical-JSON discipline.** Top-level lexicographic id keys; fixed per-entry key order; `{}` for the empty catalog (`encrypt-empty` pins the 68-byte minimal blob).
- **Forward-compat tolerance (§ 9.10.2).** `decrypt-unknown-field-tolerated` pins that an unknown additive per-entry field is ignored, not rejected — the load-bearing schema-evolution hook for v0.1.x additive fields.
- **AAD binds account identity.** `decrypt-fail-cross-vault` pins that a blob sealed under vault A's `account_number_hash` fails under vault B's.
- **Tamper rejection + non-distinguishing surface.** The two `decrypt_aead_fail` cases MUST be indistinguishable from each other (spec § 6.4).
- **Distinguishable structural rejects (MYND-227, `parse-fail-*`).** Each `CatalogBlobError` decode reject pinned individually; the `ciphertext-len-overflow` case pins the 16 MiB + 32 cap from the read direction with a 34-byte fixture (header advertises `0xffffffff` without carrying it), and `plaintext_parse_failed` is distinct from `CannotDecrypt` (the tag check already proved key possession).

## Regeneration

```bash
cargo test -p protocol --lib emit_attachments_catalog_vectors_for_paste -- --ignored
cat /tmp/myne_attachments_catalog_vectors.json
# copy the full JSON verbatim into cases.json
```

The regenerator lives in `protocol/src/attachments_catalog.rs`; it is `#[ignore]`-gated so it never runs in default `cargo test` invocations. It emits the **whole file** (fully deterministic inputs) — diff against the shipped `cases.json` and confirm existing cases regenerate byte-identically before pasting.

The MYND-227 `parse_fail` family (7 cases) has its own append generator (the original stays byte-stable — vector_id immutability):

```bash
cargo test -p protocol --lib emit_catalog_parse_fail_vectors -- --ignored
# splice the JSON array fragment at /tmp/myne_attachments_catalog_parse_fail_vectors.json into the cases array
```

## What this does NOT cover

- **Encode-only error modes.** Encode-side `CiphertextLenOverflow` (oversized plaintext at the 16-MiB cap) and `RandomnessUnavailable` cannot be expressed as decode-a-blob cases; the cap IS pinned from the read direction by `parse-fail-ciphertext-len-overflow` (MYND-227).
- **`read_or_empty` collapse behavior** — the missing-file → `{}` fallback is a host-layer call-site policy; the vectors pin only the codec underneath it.
- **Per-attachment blob bytes** — the sibling `attachment-blob` category.
- **The Tauri-layer attachments library UI + ingest flow** (MYND-164 desktop side) — verified by `src-tauri` integration tests, not cross-implementation vectors.
