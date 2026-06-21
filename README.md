# myne-test-vectors

Language-agnostic **JSON test vectors** for the [Myne](https://github.com/myne-md/myne) cryptographic protocol.

For the protocol specification, see [`docs/spec/protocol.md`](https://github.com/myne-md/myne/blob/main/docs/spec/protocol.md) in the main repository.

## Status

**Phase 1, twenty-six v0.1 categories landed.** The original nine planned categories plus `vault-search` (the `MYS1` / `vault.search` codec, MYND-29), `password-policy` (the `validate_min_length` master-password floor), `bip39-recovery` (the end-to-end recovery composition), and `aad-construction` (per-call-site AAD framing decomposition) — which closed the remaining implementation-vs-vectors gaps — plus `damm32-table` (the frozen Damm-32 quasigroup table + checksum orientation cases, MYND-74), the attachments pair `attachment-blob` (`MYA1`) + `attachments-catalog` (`MYC1`, MYND-164), `note-template` (the deterministic `{{token}}` variable engine, MYND-227 — which also closed the structural `parse_fail` coverage gap across `vault-meta` / `vault-prefs` / `attachments-catalog` / `myn-blob` / `attachment-blob`, bringing every blob family's reject surface to `vault-search` parity), the §10.3 coverage-gap pair `vault-workspace` (`MYW1` / `vault.workspace` codec) + `backup-container` (`MYB1` / `.myne-backup` container, MYND-254 — the first category whose reference implementation lives in the main repo's `src-tauri/`, not the `protocol` crate), the Saved Views pair `property-coercion` + `vault-indices` (`MYI1`, MYND-271), and `markdown-comment` (the dialect `%%…%%` editor-only comment — recognition + exclusion + terminated-vs-unterminated + strip, MYND-315; a TS-runner category like `property-coercion`), plus the two audit-gap closures `vault-dictionary` (`MYD1` / `vault.dictionary` personal-dictionary codec, MYND-266) + `vault-quickunlock` (`MYQ1` / `vault.quickunlock` device-local credential codec, MYND-329) — implemented on-disk AEAD formats that previously had no byte-level vectors, each landed in lockstep with its `hkdf-sha256` / `ctx-aead` / `aad-construction` family extensions (and, for quick-unlock, the `argon2id` `quickunlock-pin` derivation path), and `attachment-text` (the `MYE1` per-attachment extracted-text cache codec, MYND-328 — the search-attachment-content feature; landed in lockstep with the `hkdf-sha256` `attachment-text-key` role + the `ctx-aead`/`aad-construction` `aad_attachment_text_blob` builder), and `d1-clip-serializer` (the dialect **clip serializer**'s canonical-D1 emit — the `@lezer/markdown`-tree → canonical-D1 `serializeClipMarkdown` re-authors the browser extension's best-effort markdown; a TS-runner category like `property-coercion` / `markdown-comment`, browser-extension epic capture-extraction sub-feature, SYNC-7 + SYNC-11). Each landed category has a JSON-Schema-validated `cases.json` plus a domain-semantic `schema.md`. **No categories remain planned.** The canonical machine-readable contract lives at [`schema/v0.1.json`](schema/v0.1.json) and dispatches per-category sub-schemas under [`schema/v0.1/`](schema/v0.1/).

### Landed (v0.1)

- **[`vectors/phrase_normalize/`](vectors/phrase_normalize/)** — `protocol::phrase::normalize_user_phrase` NFKD + ASCII-lowercase + whitespace pipeline.
- **[`vectors/vault-prefs/`](vectors/vault-prefs/)** — `MYP1` encrypted-preferences blob codec (`encode_prefs_blob_raw` / `decode_prefs_blob`), with encrypt + decrypt-success + non-distinguishing AEAD-failure cases, grown additively per feature (MYND-183/110/191/184/185/205/58), plus the MYND-227 `parse_fail` family: one case per distinguishable `PrefsBlobError` decode reject (six pre-AEAD structural + the post-AEAD `plaintext_parse_failed`), with real-key fixtures so a port that skips a structural check surfaces as Ok instead of a coincidental AEAD error.
- **[`vectors/vault_meta_rewrite/`](vectors/vault_meta_rewrite/)** — `rewrite_vault_meta_password_side` password-side rewrite producing the 345-byte `MYV1` blob.
- **[`vectors/hkdf-sha256/`](vectors/hkdf-sha256/)** — `derive_subkey` canonical HKDF-SHA-256 helper exercising every registered `SubkeyRole` (`itemsKey-wrap`, `prefs-key`, `search-index-key`, `attachments-catalog-key`, `attachments-id-hmac`, `session-state-key`, `personal-dictionary-key`, `vault-indices-key`).
- **[`vectors/vault-meta/`](vectors/vault-meta/)** — `serialize_vault_meta` + `parse_vault_meta` create-time `MYV1` encoder/decoder pair: 345-byte round-trip + 6 structural-parse-reject cases (magic / reserved / wrong-length / protocol-version / kdf-id / argon2-version tampers — the latter four MYND-227; the KDF-byte pair pins that `parse_kdf_params` validation happens at parse time). Distinct from `vault_meta_rewrite` (which pins only the password-side rewrite).
- **[`vectors/account-number/`](vectors/account-number/)** — `parse_account_number` Crockford-base32 + Damm-32 + liberal-parsing normalization: parse permutations (canonical/display/lowercase/whitespace/prefix-optional) + strict-reject (non-Crockford char, wrong length) + Damm-32 single-substitution + adjacent-transposition catches.
- **[`vectors/myn-blob/`](vectors/myn-blob/)** — `encode_new_note_blob_raw` + `encode_update_blob_raw` + `decode_note` `MY01` per-note blob codec. 18 cases: 3 encrypt (10 B / 1 KB / 100 KB) + 3 `encrypt_update` (§6.3 note-update: same-size / grow / shrink, each pinning the verbatim 112-byte `header_prefix` splice so `enc_item_key` never re-wraps) + 2 `decrypt_ok` (1 KB round-trip + post-update round-trip) + 2 `decrypt_aead_fail` (ciphertext-tamper + the MYND-227 items-key-id-mismatch case) + 8 MYND-227 `parse_fail` structural rejects (incl. the §10.1 strict-trailing trailing-byte case; all collapse to the non-distinguishing `CannotDecrypt`, real-key fixtures). `content_type` is hardcoded `text/markdown` in v0.1.
- **[`vectors/ctx-aead/`](vectors/ctx-aead/)** — `aead_seal_ctx` + `aead_open_ctx` (XChaCha20-Poly1305 + CTX commitment outer per Chan & Rogaway 2022) + every pub `aad_*` builder's canonical byte assembly (`aad_password_wrap`, `aad_recovery_wrap`, `aad_vault_items_key_wrap`, `aad_note_item_key_wrap`, `aad_note_content_encrypt`, `aad_prefs_blob`, `aad_search_index_blob`, `aad_workspace_blob`, `aad_vault_indices_blob` — all 9 pub builders pinned). Note: this category was tracked in the prior planned list as `xchacha20-poly1305-ctx`; landed under `ctx-aead` (same scope).
- **[`vectors/argon2id/`](vectors/argon2id/)** — `derive_password_wrap_key` + `derive_recovery_wrap_key_from_normalized` at canonical Phase 1 parameters (m=256 MiB, t=3, p=2, v=0x13). Password + recovery paths × {short ASCII, long ASCII, unicode}. **Slow tests** (~3s per case at canonical params; ~18s total category runtime).
- **[`vectors/vault-search/`](vectors/vault-search/)** — `encode_search_index_blob_raw` + `decode_search_index_blob` `MYS1` `vault.search` encrypted search-index codec (MYND-29, spec §9.6). 12 cases: 3 encrypt (empty / small / 1-KB opaque inner plaintext) + decrypt round-trip + 2 non-distinguishing AEAD-failure (CTX-tag tamper + cross-vault `account_number_hash`) + 6 distinguishable structural `parse_fail` (one per `SearchBlobError` reject — bad magic / protocol_version / `index_format_version` / header-too-short / truncated / `ciphertext_len` overflow). KEK is `derive_subkey(root_key, SubkeyRole::SearchIndexKey)` (role `search-index-key`); `account_number_hash` binds via `aad_search_index_blob` (role `search-index-blob` — a **distinct** string from the KEK role). The third member of the singleton-blob family alongside `vault-prefs` (MYP1) and `myn-blob` (MY01).
- **[`vectors/password-policy/`](vectors/password-policy/)** — `validate_min_length` master-password minimum-length floor (`MIN_PASSWORD_LEN = 12`). A **boolean accept/reject conformance** category (not byte-output): counts Unicode scalar values (`str::chars().count()`), NOT UTF-8 bytes. 7 cases pinning the boundary (11 reject / 12 accept / 13 accept / empty reject) plus the load-bearing USV-vs-byte discriminators (`á`×12 = 12 USV / 24 byte accept; `á`×11 = 11 USV / 22 byte reject; 13-USV mixed-script). Guards independent clients (incl. the frontend TS hand-mirror) against a byte-length gate.
- **[`vectors/bip39-recovery/`](vectors/bip39-recovery/)** — end-to-end recovery composition `normalize_user_phrase` (§7.4) → `derive_recovery_wrap_key_from_normalized` (§3.1): a raw 24-word phrase is NFKD-normalized then fed **directly** to Argon2id (NOT through BIP-39's PBKDF2 step). 3 cases type the canonical all-zero phrase three ways (canonical / mixed-case+messy-whitespace / NFKD-foldable full-width+exotic-whitespace); all three derive the **identical** `recovery_wrap_key`, pinning the NFKD-before-Argon2id ordering. **Slow** (~3s/case, default-on like `argon2id`).
- **[`vectors/aad-construction/`](vectors/aad-construction/)** — per-call-site AAD framing decomposition for all 9 pub `aad_*` builders (incl. `aad_vault_indices_blob`, MYND-271). Pins the full AAD bytes (== `ctx-aead`'s `aad_canonical_bytes` pins) **plus** the structural segment breakdown (`aad_prefix` / `protocol_version` / `role_length_be16` / `role_label` / bound params) with byte offsets — catching a port that emits the right total but frames it wrong (dropped `u16` length-prefix, wrong field order). 9 cases (one per builder); the harness asserts every builder has one.
- **[`vectors/damm32-table/`](vectors/damm32-table/)** — the frozen Damm-32 order-32 quasigroup table (`account-number.md` §4.2) as cross-impl ground truth (MYND-74). 4 cases: 1 `table` (the full 1024-byte row-major table as hex) + 3 `checksum` (left-fold-from-0 orientation pins). Complements `account-number` (which pins `parse_account_number` behavior); this pins the table + raw index checksum beneath it.
- **[`vectors/attachment-blob/`](vectors/attachment-blob/)** — `encode_attachment_blob_raw` + `decode_attachment_blob` + `compute_attachment_id` `MYA1` per-attachment blob codec (MYND-164, spec §9.9). 18 cases across a 6-way `case_kind`: 2 encrypt + 1 decrypt round-trip + 3 non-distinguishing AEAD-failure (tamper + cross-vault + the MYND-227 items-key-id-mismatch case) + 2 `aad_construction` (both attachment AAD builders) + 2 `id_determinism` (content-addressed HMAC-SHA-256 attachment-ids keyed by `DeriveSubkey(root, 'attachments-id-hmac')`) + 8 MYND-227 `parse_fail` structural rejects (non-distinguishing `CannotDecrypt`, real-key fixtures). Key lineage: itemsKey → attachment_key → file bytes (mirrors `myn-blob`, not the root-subkey singletons).
- **[`vectors/attachments-catalog/`](vectors/attachments-catalog/)** — `encode_attachments_catalog_raw` + `decode_attachments_catalog` `MYC1` `vault.attachments` catalog codec (MYND-164, spec §9.10). 14 cases: 2 encrypt (incl. empty `{}`) + 2 decrypt_ok (round-trip + the §9.10.2 forward-compat unknown-field tolerance case) + 2 non-distinguishing AEAD-failure + 1 `aad_construction` + 7 MYND-227 `parse_fail` (one per distinguishable `CatalogBlobError` decode reject incl. the post-AEAD `plaintext_parse_failed`). KEK is `derive_subkey(root_key, SubkeyRole::AttachmentsCatalogKey)`; canonical-JSON inner plaintext (id keys sorted, fixed per-entry key order).
- **[`vectors/note-template/`](vectors/note-template/)** — `protocol::template::render` + `supported_tokens`, the deterministic note-template variable engine (MYND-188 engine, vectored with MYND-227). Not a codec and not cryptographic — deterministic cross-client behavior in the `password-policy` class: one left-to-right pass over `{{token}}` placeholders against a closed case-sensitive 5-token registry (`date`, `time`, `title`, `datetime`, `weekday`); unknown tokens / format-arg shapes / unbalanced braces pass verbatim (the escape mechanism); substituted values are never re-scanned; no clock, no locale, no unicode normalization. 12 cases: 11 `render` (incl. the load-bearing single-pass `render-title-not-rescanned` pin and an NFD-unicode byte-transparency case) + 1 `token_registry` (`supported_tokens()` order/set). All strings pinned as `*_utf8_hex`.
- **[`vectors/vault-workspace/`](vectors/vault-workspace/)** — `encode_workspace_blob_raw` + `decode_workspace_blob` `MYW1` `vault.workspace` encrypted editor-workspace-state codec (MYND-254; codec landed MYND-166, spec §9.7). 12 cases: 3 encrypt (empty / small pane-tree descriptor / 1-KB opaque inner plaintext) + decrypt round-trip + 2 non-distinguishing AEAD-failure (CTX-tag tamper + cross-vault `account_number_hash`) + 6 distinguishable structural `parse_fail` (one per `WorkspaceBlobError` reject — bad magic / protocol_version / `workspace_format_version` / header-too-short / truncated / `ciphertext_len` overflow). KEK is `derive_subkey(root_key, SubkeyRole::SessionStateKey)` (role `session-state-key`); `account_number_hash` binds via `aad_workspace_blob` (role `workspace-blob` — a **distinct** string from the KEK role). The fourth member of the singleton-blob family alongside `vault-prefs` (MYP1), `vault-search` (MYS1) and `myn-blob` (MY01).
- **[`vectors/property-coercion/`](vectors/property-coercion/)** — the Saved Views property value-typing / coercion ruleset (`markdown-dialect.md` §3.6.4, MYND-271): how a YAML 1.2 frontmatter value coerces to exactly one Myne type — `string` / `number` / `date` / `list` / `checkbox`. A deterministic, non-crypto conformance category in the `password-policy` / `note-template` class; the **first category whose reference implementation is TypeScript** (the property-index sub-feature), not the `protocol` crate. 31 cases (26 `scalar` + 5 `list`) pinning the load-bearing tie-breaks (YAML-1.2-core booleans, the quoting-demotes-checkbox/number-but-not-date asymmetry, bare-integer-is-number-not-date, no number/date reformatting, NFD byte-transparency). Hand-curated; asserted TS-side.
- **[`vectors/vault-indices/`](vectors/vault-indices/)** — `encode_vault_indices_blob_raw` + `decode_vault_indices_blob` `MYI1` `vault.indices` encrypted structured-metadata codec (MYND-271; design MYND-31, spec §9.8). Opaque inner plaintext (the desktop layer's serialized structured index — wikilink graph + tag index + title map + typed property records); KEK `derive_subkey(root_key, SubkeyRole::VaultIndicesKey)` (role `vault-indices-key`); `account_number_hash` binds via `aad_vault_indices_blob` (role `vault-indices-blob`). **Distinct blob family from `vault-search`** — `MYI1` != `MYS1`, one magic + one KEK + one AAD role each. 12 cases (the `vault-search`/`vault-workspace` shape): 3 encrypt + 1 decrypt_ok + 2 non-distinguishing decrypt_aead_fail (real-key fixtures) + 6 distinguishable parse_fail (incl. the load-bearing `format_unsupported` forward hook, §9.8.4). The fifth member of the singleton-blob family alongside `vault-prefs`/`vault-search`/`vault-workspace`/`myn-blob`.
- **[`vectors/markdown-comment/`](vectors/markdown-comment/)** — the Myne dialect `%%…%%` editor-only comment (`markdown-dialect.md` §3.1 block / §3.2 inline / §3.7.5 exclusion / §6 strip, MYND-315): which `%%` runs a conformant client recognizes as a comment and what it strips on a lossy projection (HTML / PDF / stripped-TXT / document statistics). A deterministic, non-crypto conformance category in the `property-coercion` / `note-template` class; a **TS-runner category** (the recognizer + strip live TS-side in the dialect layer, not the `protocol` crate). 14 cases via a 3-way `case_kind`: `recognize` (7 — inline/block/multiline/empty/multiple/mixed), `inert` (3 — the load-bearing **unterminated-`%%`-is-inert** tie-breaks + single-`%`), `exclusion` (4 — `%%` inside fenced code / inline code / wikilink / inline-math is literal). Pins `expected_has_comment` + the byte-exact strip output (`.md` is exempt — byte-verbatim, SYNC-2). Hand-curated; asserted TS-side.
- **[`vectors/vault-dictionary/`](vectors/vault-dictionary/)** — `encode_dictionary_blob_raw` + `decode_dictionary_blob` `MYD1` `vault.dictionary` encrypted personal-dictionary codec (MYND-266, spec §9.13). Opaque inner plaintext (the desktop layer's serialized `{personal_words, ignore_words}` word sets); KEK `derive_subkey(root_key, SubkeyRole::PersonalDictKey)` (role `personal-dictionary-key`); `account_number_hash` binds via `aad_personal_dictionary_blob` (role `personal-dictionary-blob`). The sixth member of the vault-singleton blob family alongside `vault-prefs`/`vault-search`/`vault-workspace`/`vault-indices`/`myn-blob`. 12 cases (the `vault-indices` shape): 3 encrypt + 1 decrypt_ok + 2 non-distinguishing decrypt_aead_fail (real-key fixtures) + 6 distinguishable parse_fail (incl. the load-bearing `format_unsupported` forward hook, §9.13.4). Landed in lockstep with the `hkdf-sha256` `personal-dictionary-key` role + the `ctx-aead`/`aad-construction` `aad_personal_dictionary_blob` builder.
- **[`vectors/vault-quickunlock/`](vectors/vault-quickunlock/)** — `encode_quick_unlock_blob_raw` + `decode_quick_unlock_blob` `MYQ1` `vault.quickunlock` device-local credential codec (MYND-329, spec §9.14). Wraps a **third copy of `root_key`** under a provider-supplied `wrap_key` (biometric / PIN); the codec is provider-agnostic and never runs Argon2id — so vectors supply `wrap_key` directly. **No HKDF SubkeyRole** (a root_key-derived KEK would be circular); only the AAD role `quick-unlock-blob`, with `provider_kind` (0x01/0x02) AAD-bound. Fixed 64-byte ciphertext (no size cap). 12 cases: 2 encrypt (biometric + PIN) + 1 decrypt_ok + 3 decrypt_aead_fail (tag-tamper + cross-vault + provider-flip, all non-distinguishing) + 6 parse_fail. The PIN→`wrap_key` Argon2id derivation is pinned separately as the `argon2id` `quickunlock-pin` path. Landed in lockstep with the `ctx-aead`/`aad-construction` `aad_quick_unlock_blob` builder + the `argon2id` `quickunlock-pin` derivation path.
- **[`vectors/attachment-text/`](vectors/attachment-text/)** — `encode_attachment_text_blob_raw` + `decode_attachment_text_blob` `MYE1` per-attachment extracted-text cache codec (MYND-328, spec §9.15 — the search-attachment-content feature). Stored at `attachments-text/<attachment-id-hex>`; opaque inner plaintext (the desktop's postcard `ExtractedText` — PDF/Word/text extraction; no canonical-JSON, like vault-search/dictionary). KEK `derive_subkey(root_key, SubkeyRole::AttachmentTextKey)` (role `attachment-text-key`) and AAD `aad_attachment_text_blob` (role `attachment-text-blob`) which **additionally binds `attachment_id` + `extractor_version`** — so one KEK is safe across all attachment-text blobs. 13 cases: 3 encrypt + 1 decrypt_ok + 3 decrypt_aead_fail (tag-tamper + cross-vault + **wrong-attachment-id**, all non-distinguishing §6.4) + 6 `parse_fail` (incl. the load-bearing `extractor_version_unsupported` forward hook, §9.15.4). Generator: `emit_attachment_text_vectors_for_paste` (`attachment_text_blob.rs`).
- **[`vectors/backup-container/`](vectors/backup-container/)** — the `MYB1` `.myne-backup` whole-vault backup container (MYND-254; codec landed MYND-217, spec §9.11): deterministic write side, credential-free 5-stage verifier (stages 1–4), and the §9.11.3 closed-allowlist path classifier. An **unkeyed** integrity layer — per-member SHA-256 over `path_len_be16 || path || member_bytes` detects accidental corruption, never tampering. 32 cases: 2 `encode` (minimal + full-allowlist; unsorted input pins writer-sorts canonical raw-byte order) + 2 `verify_ok` + 14 `verify_fail` (all 13 vectorable error variants + the `MYB5` v005-reservation magic; the two ceiling cases pin the §9.11.6 stage order) + 14 `classify_path` (one accept per family + 8 load-bearing rejects). **The first category whose reference implementation, generator and runner live in the main repo's `src-tauri/` (package `myne-desktop`), not the `protocol` crate** — the codec is std/fs-bound by design.
- **[`vectors/d1-clip-serializer/`](vectors/d1-clip-serializer/)** — the Myne dialect **clip serializer** `serializeClipMarkdown` (`src/dialect/clip.ts`, the `@lezer/markdown`-tree → canonical-D1 emitter; `markdown-dialect.md` §5/§6, browser-extension epic capture-extraction sub-feature, SYNC-7 + SYNC-11). Each case is a Defuddle-produced markdown source → byte-exact canonical-D1 output. A clip is AUTHORED content (not a C1 byte-preserved import), so the serializer renormalizes recognized-on-parse non-canonical forms; every pinned output is a serializer fixed point (round-trip + idempotent). A deterministic, non-crypto conformance category in the `property-coercion` / `markdown-comment` / `note-template` class; a **TS-runner category** (the serializer lives TS-side, no Rust generator). 35 cases via a 5-way `case_kind` — `floor` (faithful table / code / inline-code), `math` (`$…$` / `$$…$$` against D1's `$`-tokenizer + the degrade-to-inert currency guard — task 12 extended this subset to 15 math cases), `wedge` (headings / lists / links / emphasis / callouts / tasks / footnotes / highlights), `normalize` (Setext→ATX, ref-link→inline, alt-emphasis-delims, indented-code→fenced), `degrade` (a no-emit-form construct dropped, never broken markdown). Hand-curated; asserted TS-side.

### Planned

**None — every category that was planned has landed.** `password-policy`, `bip39-recovery`, and `aad-construction` (the last two previously listed here) closed in this round; `aad-construction` is framed as a decomposition layer atop `ctx-aead`'s existing builder-byte pins rather than a duplicate. The latest landing is `d1-clip-serializer` (the clip serializer's canonical-D1 emit); the capture-extraction task-12 math subset extends it in place rather than adding a new category.

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
    ├── aad-construction.json
    ├── damm32-table.json
    ├── attachment-blob.json
    ├── attachments-catalog.json
    ├── note-template.json
    ├── vault-workspace.json
    ├── backup-container.json
    ├── property-coercion.json
    ├── vault-indices.json
    ├── markdown-comment.json
    ├── vault-dictionary.json
    ├── vault-quickunlock.json
    ├── attachment-text.json
    └── d1-clip-serializer.json
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
  -r schema/v0.1/note-template.json \
  -r schema/v0.1/vault-workspace.json \
  -r schema/v0.1/backup-container.json \
  -r schema/v0.1/property-coercion.json \
  -r schema/v0.1/vault-indices.json \
  -r schema/v0.1/markdown-comment.json \
  -r schema/v0.1/vault-dictionary.json \
  -r schema/v0.1/vault-quickunlock.json \
  -r schema/v0.1/attachment-text.json \
  -r schema/v0.1/d1-clip-serializer.json \
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
  -d "vectors/note-template/v0.1/cases.json" \
  -d "vectors/vault-workspace/v0.1/cases.json" \
  -d "vectors/backup-container/v0.1/cases.json" \
  -d "vectors/property-coercion/v0.1/cases.json" \
  -d "vectors/vault-indices/v0.1/cases.json" \
  -d "vectors/markdown-comment/v0.1/cases.json" \
  -d "vectors/vault-dictionary/v0.1/cases.json" \
  -d "vectors/vault-quickunlock/v0.1/cases.json" \
  -d "vectors/attachment-text/v0.1/cases.json" \
  -d "vectors/d1-clip-serializer/v0.1/cases.json" \
  --spec=draft2020
```

Every `cases.json` must validate clean. Byte-level conformance against the live Rust impl is verified separately via the integration tests in the main repo at `protocol/tests/vectors.rs` (`cargo test -p protocol --test vectors`) — except `backup-container`, whose runner lives in `src-tauri/src/backup_container.rs` (`cargo test -p myne-desktop --lib backup_container_vectors`).

## Why language-agnostic JSON

Any independent implementation of the Myne protocol — in Rust, Go, Swift, Kotlin, JavaScript, or anything else — must produce bit-identical outputs from these inputs. A reviewer or contributor verifying a port should not need to read the reference Rust implementation; running the test runner of their language against this repository should be enough.

## License

[CC0 1.0](LICENSE) — public domain. Test vectors are reference material for independent implementations; they should be maximally reusable.
