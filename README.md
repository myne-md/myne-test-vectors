# myne-test-vectors

Language-agnostic **JSON test vectors** for the [Myne](https://github.com/myne-md/myne) cryptographic protocol.

For the protocol specification, see [`docs/spec/protocol.md`](https://github.com/myne-md/myne/blob/main/docs/spec/protocol.md) in the main repository.

## Status

**Phase 1, sixteen v0.1 categories landed.** The original nine planned categories plus `vault-search` (the `MYS1` / `vault.search` codec, MYND-29), `password-policy` (the `validate_min_length` master-password floor), `bip39-recovery` (the end-to-end recovery composition), and `aad-construction` (per-call-site AAD framing decomposition) — which closed the remaining implementation-vs-vectors gaps — plus `damm32-table` (the frozen Damm-32 quasigroup table + checksum orientation cases, MYND-74) and the attachments pair `attachment-blob` (`MYA1`) + `attachments-catalog` (`MYC1`, MYND-164). Each landed category has a JSON-Schema-validated `cases.json` plus a domain-semantic `schema.md`. **No categories remain planned.** The canonical machine-readable contract lives at [`schema/v0.1.json`](schema/v0.1.json) and dispatches per-category sub-schemas under [`schema/v0.1/`](schema/v0.1/).

### Landed (v0.1)

- **[`vectors/phrase_normalize/`](vectors/phrase_normalize/)** — `protocol::phrase::normalize_user_phrase` NFKD + ASCII-lowercase + whitespace pipeline.
- **[`vectors/vault-prefs/`](vectors/vault-prefs/)** — `MYP1` encrypted-preferences blob codec (`encode_prefs_blob_raw` / `decode_prefs_blob`), with encrypt + decrypt-success + non-distinguishing AEAD-failure cases.
- **[`vectors/vault_meta_rewrite/`](vectors/vault_meta_rewrite/)** — `rewrite_vault_meta_password_side` password-side rewrite producing the 345-byte `MYV1` blob.
- **[`vectors/hkdf-sha256/`](vectors/hkdf-sha256/)** — `derive_subkey` canonical HKDF-SHA-256 helper exercising every registered `SubkeyRole` (`itemsKey-wrap`, `prefs-key`, `search-index-key`, `attachments-catalog-key`, `attachments-id-hmac`).
- **[`vectors/vault-meta/`](vectors/vault-meta/)** — `serialize_vault_meta` + `parse_vault_meta` create-time `MYV1` encoder/decoder pair: 345-byte round-trip + structural-parse-reject cases (magic / reserved tamper). Distinct from `vault_meta_rewrite` (which pins only the password-side rewrite).
- **[`vectors/account-number/`](vectors/account-number/)** — `parse_account_number` Crockford-base32 + Damm-32 + liberal-parsing normalization: parse permutations (canonical/display/lowercase/whitespace/prefix-optional) + strict-reject (non-Crockford char, wrong length) + Damm-32 single-substitution + adjacent-transposition catches.
- **[`vectors/myn-blob/`](vectors/myn-blob/)** — `encode_new_note_blob_raw` + `encode_update_blob_raw` + `decode_note` `MY01` per-note blob codec. 9 cases: 3 encrypt (10 B / 1 KB / 100 KB) + 3 `encrypt_update` (§6.3 note-update: same-size / grow / shrink, each pinning the verbatim 112-byte `header_prefix` splice so `enc_item_key` never re-wraps) + 2 `decrypt_ok` (1 KB round-trip + post-update round-trip) + 1 ciphertext-tamper reject. `content_type` is hardcoded `text/markdown` in v0.1.
- **[`vectors/ctx-aead/`](vectors/ctx-aead/)** — `aead_seal_ctx` + `aead_open_ctx` (XChaCha20-Poly1305 + CTX commitment outer per Chan & Rogaway 2022) + every pub `aad_*` builder's canonical byte assembly (`aad_password_wrap`, `aad_recovery_wrap`, `aad_vault_items_key_wrap`, `aad_note_item_key_wrap`, `aad_note_content_encrypt`, `aad_prefs_blob`, `aad_search_index_blob` — all 7 pub builders pinned). Note: this category was tracked in the prior planned list as `xchacha20-poly1305-ctx`; landed under `ctx-aead` (same scope).
- **[`vectors/argon2id/`](vectors/argon2id/)** — `derive_password_wrap_key` + `derive_recovery_wrap_key_from_normalized` at canonical Phase 1 parameters (m=256 MiB, t=3, p=2, v=0x13). Password + recovery paths × {short ASCII, long ASCII, unicode}. **Slow tests** (~3s per case at canonical params; ~18s total category runtime).
- **[`vectors/vault-search/`](vectors/vault-search/)** — `encode_search_index_blob_raw` + `decode_search_index_blob` `MYS1` `vault.search` encrypted search-index codec (MYND-29, spec §9.6). 12 cases: 3 encrypt (empty / small / 1-KB opaque inner plaintext) + decrypt round-trip + 2 non-distinguishing AEAD-failure (CTX-tag tamper + cross-vault `account_number_hash`) + 6 distinguishable structural `parse_fail` (one per `SearchBlobError` reject — bad magic / protocol_version / `index_format_version` / header-too-short / truncated / `ciphertext_len` overflow). KEK is `derive_subkey(root_key, SubkeyRole::SearchIndexKey)` (role `search-index-key`); `account_number_hash` binds via `aad_search_index_blob` (role `search-index-blob` — a **distinct** string from the KEK role). The third member of the singleton-blob family alongside `vault-prefs` (MYP1) and `myn-blob` (MY01).
- **[`vectors/password-policy/`](vectors/password-policy/)** — `validate_min_length` master-password minimum-length floor (`MIN_PASSWORD_LEN = 12`). A **boolean accept/reject conformance** category (not byte-output): counts Unicode scalar values (`str::chars().count()`), NOT UTF-8 bytes. 7 cases pinning the boundary (11 reject / 12 accept / 13 accept / empty reject) plus the load-bearing USV-vs-byte discriminators (`á`×12 = 12 USV / 24 byte accept; `á`×11 = 11 USV / 22 byte reject; 13-USV mixed-script). Guards independent clients (incl. the frontend TS hand-mirror) against a byte-length gate.
- **[`vectors/bip39-recovery/`](vectors/bip39-recovery/)** — end-to-end recovery composition `normalize_user_phrase` (§7.4) → `derive_recovery_wrap_key_from_normalized` (§3.1): a raw 24-word phrase is NFKD-normalized then fed **directly** to Argon2id (NOT through BIP-39's PBKDF2 step). 3 cases type the canonical all-zero phrase three ways (canonical / mixed-case+messy-whitespace / NFKD-foldable full-width+exotic-whitespace); all three derive the **identical** `recovery_wrap_key`, pinning the NFKD-before-Argon2id ordering. **Slow** (~3s/case, default-on like `argon2id`).
- **[`vectors/aad-construction/`](vectors/aad-construction/)** — per-call-site AAD framing decomposition for all 7 pub `aad_*` builders. Pins the full AAD bytes (== `ctx-aead`'s `aad_canonical_bytes` pins) **plus** the structural segment breakdown (`aad_prefix` / `protocol_version` / `role_length_be16` / `role_label` / bound params) with byte offsets — catching a port that emits the right total but frames it wrong (dropped `u16` length-prefix, wrong field order). 7 cases (one per builder); the harness asserts every builder has one.
- **[`vectors/damm32-table/`](vectors/damm32-table/)** — the frozen Damm-32 order-32 quasigroup table (`account-number.md` §4.2) as cross-impl ground truth (MYND-74). 4 cases: 1 `table` (the full 1024-byte row-major table as hex) + 3 `checksum` (left-fold-from-0 orientation pins). Complements `account-number` (which pins `parse_account_number` behavior); this pins the table + raw index checksum beneath it.
- **[`vectors/attachment-blob/`](vectors/attachment-blob/)** — `encode_attachment_blob_raw` + `decode_attachment_blob` + `compute_attachment_id` `MYA1` per-attachment blob codec (MYND-164, spec §9.9). 9 cases across a 5-way `case_kind`: 2 encrypt + 1 decrypt round-trip + 2 non-distinguishing AEAD-failure (tamper + cross-vault) + 2 `aad_construction` (both attachment AAD builders) + 2 `id_determinism` (content-addressed HMAC-SHA-256 attachment-ids keyed by `DeriveSubkey(root, 'attachments-id-hmac')`). Key lineage: itemsKey → attachment_key → file bytes (mirrors `myn-blob`, not the root-subkey singletons).
- **[`vectors/attachments-catalog/`](vectors/attachments-catalog/)** — `encode_attachments_catalog_raw` + `decode_attachments_catalog` `MYC1` `vault.attachments` catalog codec (MYND-164, spec §9.10). 7 cases: 2 encrypt (incl. empty `{}`) + 2 decrypt_ok (round-trip + the §9.10.2 forward-compat unknown-field tolerance case) + 2 non-distinguishing AEAD-failure + 1 `aad_construction`. KEK is `derive_subkey(root_key, SubkeyRole::AttachmentsCatalogKey)`; canonical-JSON inner plaintext (id keys sorted, fixed per-entry key order).

### Planned

**None — every category that was planned has landed.** `password-policy`, `bip39-recovery`, and `aad-construction` (the last two previously listed here) closed in this round; `aad-construction` is framed as a decomposition layer atop `ctx-aead`'s existing builder-byte pins rather than a duplicate.

Each category pins the protocol version (`v0.1` for the initial set) so that future major-version vectors (`v005` and beyond) can coexist without ambiguity.

## Schema layout

```
schema/
├── v0.1.json                          ← root schema (top-level shape + oneOf dispatch on `schema` field)
└── v0.1/
    ├── phrase_normalize.json          ← per-category case shape
    ├── vault-prefs.json
    ├── vault_meta_rewrite.json
    ├── hkdf-sha256.json
    ├── vault-meta.json
    ├── account-number.json
    ├── myn-blob.json
    ├── ctx-aead.json
    ├── argon2id.json
    ├── vault-search.json
    ├── password-policy.json
    ├── bip39-recovery.json
    └── aad-construction.json
```

Adding a new category requires landing its sub-schema file under `schema/v0.1/` and extending both the `schema` enum and the `oneOf` branches in the root schema in lockstep.

## Validating vectors locally

Schema validation is currently a developer-side step — CI integration is deferred. Run from this repository's root:

```bash
npx -y ajv-cli@5 validate \
  -s schema/v0.1.json \
  -r schema/v0.1/phrase_normalize.json \
  -r schema/v0.1/vault-prefs.json \
  -r schema/v0.1/vault_meta_rewrite.json \
  -r schema/v0.1/hkdf-sha256.json \
  -r schema/v0.1/vault-meta.json \
  -r schema/v0.1/account-number.json \
  -r schema/v0.1/myn-blob.json \
  -r schema/v0.1/ctx-aead.json \
  -r schema/v0.1/argon2id.json \
  -r schema/v0.1/vault-search.json \
  -r schema/v0.1/password-policy.json \
  -r schema/v0.1/bip39-recovery.json \
  -r schema/v0.1/aad-construction.json \
  -r schema/v0.1/damm32-table.json \
  -r schema/v0.1/attachment-blob.json \
  -r schema/v0.1/attachments-catalog.json \
  -d "vectors/phrase_normalize/v0.1/cases.json" \
  -d "vectors/vault-prefs/v0.1/cases.json" \
  -d "vectors/vault_meta_rewrite/v0.1/cases.json" \
  -d "vectors/hkdf-sha256/v0.1/cases.json" \
  -d "vectors/vault-meta/v0.1/cases.json" \
  -d "vectors/account-number/v0.1/cases.json" \
  -d "vectors/myn-blob/v0.1/cases.json" \
  -d "vectors/ctx-aead/v0.1/cases.json" \
  -d "vectors/argon2id/v0.1/cases.json" \
  -d "vectors/vault-search/v0.1/cases.json" \
  -d "vectors/password-policy/v0.1/cases.json" \
  -d "vectors/bip39-recovery/v0.1/cases.json" \
  -d "vectors/aad-construction/v0.1/cases.json" \
  -d "vectors/damm32-table/v0.1/cases.json" \
  -d "vectors/attachment-blob/v0.1/cases.json" \
  -d "vectors/attachments-catalog/v0.1/cases.json" \
  --spec=draft2020
```

Every `cases.json` must validate clean. Byte-level conformance against the live Rust impl is verified separately via the integration tests in the main repo at `protocol/tests/vectors.rs` (`cargo test -p protocol --test vectors`).

## Why language-agnostic JSON

Any independent implementation of the Myne protocol — in Rust, Go, Swift, Kotlin, JavaScript, or anything else — must produce bit-identical outputs from these inputs. A reviewer or contributor verifying a port should not need to read the reference Rust implementation; running the test runner of their language against this repository should be enough.

## License

[CC0 1.0](LICENSE) — public domain. Test vectors are reference material for independent implementations; they should be maximally reusable.
