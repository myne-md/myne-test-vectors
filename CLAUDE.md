# myne-test-vectors — JSON test vectors for Myne v0.1 protocol

> Companion repository to [`myne`](https://github.com/myne-md/myne). Public, **CC0** (public domain) — vectors must remain maximally reusable for independent implementations in any language.

## Purpose

Language-agnostic JSON test vectors for the Myne cryptographic protocol. Any independent implementation (Rust, Go, Swift, Kotlin, JavaScript, etc.) MUST produce **bit-identical outputs** from these inputs. A reviewer or contributor verifying a port should not need to read the reference Rust implementation; running their language's test runner against this repository should be enough.

## Status

**Phase 1, twenty-five v0.1 categories landed.** The `schema/v0.1.json` root schema + per-category sub-schemas under `schema/v0.1/` are the authoritative machine-readable contract. Landed: `phrase_normalize`, `vault-prefs`, `vault_meta_rewrite`, `hkdf-sha256`, `vault-meta`, `account-number`, `myn-blob`, `ctx-aead`, `argon2id`, `vault-search`, `password-policy`, `bip39-recovery`, `aad-construction`, `damm32-table`, `attachment-blob`, `attachments-catalog`, `note-template`, `vault-workspace`, `backup-container`, `property-coercion`, `vault-indices`, `markdown-comment`, `vault-dictionary`, `vault-quickunlock`, `attachment-text`. `attachment-text` (MYND-328, §9.15, `MYE1`) is the per-attachment extracted-text cache codec for the search-attachment-content feature — opaque inner plaintext like vault-search/dictionary, but its `aad_attachment_text_blob` additionally binds `attachment_id` (so one KEK is safe across all such blobs) + `extractor_version`; landed in lockstep with the `hkdf-sha256` `attachment-text-key` role + the `ctx-aead`/`aad-construction` `aad_attachment_text_blob` builder. `vault-dictionary` (MYND-266, §9.13, `MYD1`) + `vault-quickunlock` (MYND-329, §9.14, `MYQ1`) closed the two audit-found coverage gaps — both implemented on-disk AEAD formats that previously had no byte-level vectors; each landed in lockstep with its `hkdf-sha256` / `ctx-aead` / `aad-construction` family extensions (`vault-quickunlock` has no HKDF role — only an AAD role + the `argon2id` `quickunlock-pin` derivation path). `markdown-comment` (MYND-315) pins the dialect `%%…%%` editor-only comment (recognition + exclusion zones + terminated-vs-unterminated tie-break + lossy-projection strip output); like `property-coercion` it is a TS-runner category (the recognizer + strip live TS-side in the dialect layer, not the `protocol` crate). `vault-indices` (MYND-271, §9.8) is the `MYI1` structured-metadata blob codec — the structural twin of `vault-search`/`vault-workspace`, landed in lockstep with the `hkdf-sha256` `vault-indices-key` role + the `ctx-aead`/`aad-construction` `aad_vault_indices_blob` builder. `property-coercion` (MYND-271, Saved Views) pins the frontmatter-property value-typing / coercion ruleset (`markdown-dialect.md` §3.6.4) — a deterministic, non-crypto, hand-curated category in the `password-policy` / `note-template` class whose runner is TS-side (the coercion lives in the property-index sub-feature, not the Rust `protocol` crate). `password-policy` / `bip39-recovery` / `aad-construction` closed the implementation-vs-vectors gaps for surfaces unpinned by cross-impl vectors; `damm32-table` (MYND-74) pins the frozen Damm-32 quasigroup table itself as cross-impl ground truth alongside the existing `account-number` parse-behavior vectors; the attachments pair `attachment-blob` (`MYA1`) + `attachments-catalog` (`MYC1`) landed with MYND-164; `note-template` (MYND-227, the deterministic `{{token}}` engine) plus a cross-category structural `parse_fail` coverage closure (vault-meta / vault-prefs / attachments-catalog / myn-blob / attachment-blob now pin their reject surfaces at vault-search parity) landed with MYND-227; the §10.3 coverage-gap pair `vault-workspace` (`MYW1`) + `backup-container` (`MYB1`) landed with MYND-254 (`backup-container` is the first family whose generator/runner live in the main repo's `src-tauri/`, not `protocol/` — the codec is std/fs-bound). **No categories remain planned.** CI integration is currently deferred — schema validation and byte-level integration run as developer-side steps. See main repo `agent tooling/features/test-vector-schema/` (the canonical schema mechanism + first 4 categories), `agent tooling/features/lane-d-completion/` (the next 5), and `agent tooling/features/damm32-table-freeze/` (`damm32-table`) for the landing plans.

## Vector categories

### Landed (v0.1)

- `vectors/phrase_normalize/` — `protocol::phrase::normalize_user_phrase` NFKD + ASCII-lowercase + whitespace pipeline (spec §7.4).
- `vectors/vault-prefs/` — `MYP1` encrypted-preferences blob codec; encrypt + decrypt + non-distinguishing AEAD-failure + distinguishable `parse_fail` case_kinds (spec §9.5 + §6.4). Includes the MYND-205 `label` cases (NFD→NFC normalization + `item_order` < `label` < `screen_capture_protection` canonical ordering), the §9.5.2 forward-compat unknown-field tolerance case (`decrypt-unknown-field-tolerated`), the MYND-58 `last_backup_at` cases (epoch-seconds u64 bare-decimal encoding + `label` < `last_backup_at` < `screen_capture_protection` ordering), and the MYND-227 `parse_fail` family (one case per `PrefsBlobError` decode reject incl. the post-AEAD `plaintext_parse_failed`; real-key fixtures so a skipped structural check surfaces as Ok).
- `vectors/vault_meta_rewrite/` — `MYV1` password-side rewrite producing the 345-byte vault.meta (spec §9.3 + recovery-flow §4.2/§4.4). 3 cases: fast-KDF canonical + unicode-password (UTF-8 bytes fed verbatim to Argon2id, no normalization on this path) + `canonical-kdf-params` (existing-meta params are USED by the derivation, not just copied — **~3 s + 256 MiB** per runner execution for that case).
- `vectors/hkdf-sha256/` — Subkey derivations through `derive_subkey` exercising every registered `SubkeyRole` (`itemsKey-wrap`, `prefs-key`, `search-index-key`, `attachments-catalog-key`, `attachments-id-hmac`, `session-state-key`, `personal-dictionary-key`, `vault-indices-key`, `attachment-text-key`) with `myne.v0.1 ` info-prefix (spec §3.2 + §3.4). 24 cases (the MYND-328 `attachment-text-key` pair added on top of the MYND-266 `personal-dictionary-key` + MYND-271 `vault-indices-key` pairs: zero-root + alpha each); the alpha-root pairs carry `informational_info_utf8_hex` per role.
- `vectors/vault-meta/` — `serialize_vault_meta` + `parse_vault_meta` create-time MYV1 encoder/decoder pair; round-trip + 6 structural-parse-rejects (magic / reserved / wrong-length / protocol-version / kdf-id / argon2-version tampers — the latter four MYND-227; the KDF pair pins `parse_kdf_params` validation at parse time) (spec §9.3).
- `vectors/account-number/` — `parse_account_number` Crockford-base32 + Damm-32 + liberal-parsing normalization; parse permutations + strict-reject + Damm-32 substitution/transposition catches (spec §5 + decisions.md 2026-04-28 polynomial freeze).
- `vectors/myn-blob/` — `encode_new_note_blob_raw` + `encode_update_blob_raw` + `decode_note` MY01 per-note blob. 18 cases: 3 encrypt (10 B / 1 KB / 100 KB) + 3 `encrypt_update` (§6.3 note-update: same-size/grow/shrink, each pinning the verbatim 112-byte `header_prefix` splice — `enc_item_key` never re-wraps, §3.3) + 2 `decrypt_ok` (round-trip + post-update) + 2 `decrypt_aead_fail` (ciphertext-tamper + items-key-id-mismatch, MYND-227) + 8 `parse_fail` (MYND-227 structural rejects incl. §10.1 strict-trailing; non-distinguishing CannotDecrypt, real-key fixtures) (spec §9.2 + §6.3 + §6.4); `content_type` hardcoded `text/markdown`.
- `vectors/ctx-aead/` — `aead_seal_ctx` + `aead_open_ctx` AEAD primitive + every pub `aad_*` builder's canonical bytes (4-way `case_kind` including `aad_canonical_bytes`); 16 cases, 11 builders pinned individually (incl. `aad_search_index_blob`, MYND-29; `aad_workspace_blob`, MYND-254; `aad_vault_indices_blob`, MYND-271; `aad_personal_dictionary_blob`, MYND-266; `aad_quick_unlock_blob` — with its trailing `provider_kind` byte — MYND-329) (spec §5.1 + §5.2 + §6.4). Tracked in earlier planned list as `xchacha20-poly1305-ctx`; renamed `ctx-aead` on landing.
- `vectors/argon2id/` — `derive_password_wrap_key` + `derive_recovery_wrap_key_from_normalized` at canonical Phase 1 Argon2id params (m=256 MiB, t=3, p=2, v=0x13); password + recovery paths × {short, long, unicode} + the MYND-329 `quickunlock-pin` path (`derive_quick_unlock_pin_wrap_key`, §9.14.2 — the PIN run through Argon2id with the distinct `argon2id-quickunlock-pin|` salt prefix; `input_salt_random_hex` carries the 16-byte pin_salt) (spec §3 + §9.14.2 + decisions.md 2026-04-26 KDF calibration). 7 cases; **~21s runtime cost** at canonical params; vectors are default-on (not `#[ignore]`-gated). The runner pins all three derivation paths via a coverage guard.
- `vectors/vault-search/` — `encode_search_index_blob_raw` + `decode_search_index_blob` `MYS1` `vault.search` encrypted search-index codec (MYND-29 / D8, spec §9.6). Opaque inner plaintext (no canonical-JSON, unlike vault-prefs); KEK `derive_subkey(root_key, SubkeyRole::SearchIndexKey)` (role `search-index-key`) and AAD `aad_search_index_blob` (role `search-index-blob` — a **distinct** load-bearing string). 12 cases: 3 encrypt + 1 decrypt_ok + 2 decrypt_aead_fail (non-distinguishing, §6.4) + 6 `parse_fail` (distinguishable pre-decryption structural rejects). Third member of the singleton-blob family with `vault-prefs` (MYP1) + `myn-blob` (MY01).
- `vectors/password-policy/` — `validate_min_length` master-password floor (`MIN_PASSWORD_LEN = 12`); **boolean accept/reject conformance** (not byte-output). Counts USVs (`str::chars().count()`), NOT UTF-8 bytes (decisions.md 2026-05-06). 7 cases: ASCII boundary (11 reject / 12 / 13 accept) + empty + the USV-vs-byte discriminators (`á`×12 accept / `á`×11 reject / mixed-script). Single case shape, `expected_kind` discriminator (no `case_kind`).
- `vectors/bip39-recovery/` — end-to-end recovery composition `normalize_user_phrase` (§7.4) → `derive_recovery_wrap_key_from_normalized` (§3.1): raw 24-word phrase NFKD-normalized then fed **directly** to Argon2id (NOT BIP-39's PBKDF2 step). 3 cases type the canonical all-zero phrase three ways (canonical / mixed-case+whitespace / NFKD full-width+exotic-space); all derive the **identical** wrap key, pinning NFKD-before-Argon2id. **~3s/case, default-on** like `argon2id`.
- `vectors/aad-construction/` — per-call-site AAD framing decomposition for all 11 pub `aad_*` builders (incl. `aad_vault_indices_blob`, MYND-271; `aad_personal_dictionary_blob`, MYND-266; `aad_quick_unlock_blob`, MYND-329 — the only builder with a trailing `provider_kind` segment). Pins the full AAD bytes (== `ctx-aead`'s `aad_canonical_bytes` pins) **plus** the segment breakdown (`aad_prefix` / `protocol_version` / `role_length_be16` / `role_label` / bound params / `provider_kind`) with offsets — a decomposition layer atop `ctx-aead`, not a duplicate. 11 cases (one per builder); harness asserts full builder coverage.
- `vectors/damm32-table/` — the frozen Damm-32 order-32 quasigroup table (`account-number.md` §4.2) as cross-impl ground truth (MYND-74). 4 cases via a 2-way `case_kind`: 1 `table` (the full 1024-byte row-major table as hex — `table[interim][symbol]`, GF(2⁵) with x∗y=α·(x⊕y), poly `0b100101=37`, Crockford index order) + 3 `checksum` (left-fold-from-0 orientation: single-step `[1]→2`, baseline `0..18→26`, all-high `31×19`). Reference: `protocol::damm32_table()` / `damm32_step`. Complements `account-number` (which pins `parse_account_number` behavior); this pins the table + raw index checksum beneath it.
- `vectors/attachment-blob/` — `MYA1` per-attachment blob codec (MYND-164, spec §9.9): `encode_attachment_blob_raw` + `decode_attachment_blob` + `compute_attachment_id` + both attachment AAD builders. 18 cases via a 6-way `case_kind` (`encrypt`, `decrypt_ok`, `decrypt_aead_fail` incl. the MYND-227 items-key-id-mismatch case, `aad_construction`, `id_determinism`, plus 8 MYND-227 `parse_fail` structural rejects — non-distinguishing CannotDecrypt, real-key fixtures). Key lineage itemsKey → attachment_key → file bytes (mirrors `myn-blob`); content-addressed ids via HMAC-SHA-256 keyed by `DeriveSubkey(root, 'attachments-id-hmac')`.
- `vectors/attachments-catalog/` — `MYC1` `vault.attachments` catalog codec (MYND-164, spec §9.10): `encode_attachments_catalog_raw` + `decode_attachments_catalog` + `aad_attachments_catalog`. 14 cases via a 5-way `case_kind` (`encrypt` incl. empty `{}`, `decrypt_ok` incl. the §9.10.2 forward-compat unknown-field tolerance case, `decrypt_aead_fail`, `aad_construction`, plus 7 MYND-227 `parse_fail` cases — one per `CatalogBlobError` decode reject incl. the post-AEAD `plaintext_parse_failed`). KEK = `DeriveSubkey(root, 'attachments-catalog-key')`; canonical-JSON inner plaintext (sorted id keys, fixed per-entry key order, NFC strings).
- `vectors/note-template/` — `protocol::template::render` + `supported_tokens`, the deterministic note-template variable engine (MYND-188 engine, vectored with MYND-227). Not a codec and not cryptographic — deterministic cross-client behavior in the `password-policy` class. 12 cases via a 2-way `case_kind`: 11 `render` (all-five-tokens, repeated/multiline, unknown-token verbatim = the escape mechanism, case-sensitivity, format-arg literal, stray/unbalanced braces, rescan-after-literal-`{{`, empty template/title, the load-bearing single-pass `render-title-not-rescanned` pin, NFD-unicode byte-transparency) + 1 `token_registry` (`supported_tokens()` order/set). All strings pinned as `*_utf8_hex`; informational UTF-8 mirrors are runner-cross-checked.
- `vectors/vault-workspace/` — `encode_workspace_blob_raw` + `decode_workspace_blob` `MYW1` `vault.workspace` encrypted editor-workspace-state codec (MYND-254; codec landed MYND-166, spec §9.7). Opaque inner plaintext (the desktop layer's serialized pane tree — no canonical-JSON, like vault-search); KEK `derive_subkey(root_key, SubkeyRole::SessionStateKey)` (role `session-state-key`) and AAD `aad_workspace_blob` (role `workspace-blob` — a **distinct** load-bearing string). 12 cases: 3 encrypt + 1 decrypt_ok + 2 decrypt_aead_fail (non-distinguishing, §6.4) + 6 `parse_fail` (one per distinguishable pre-decryption structural reject incl. the load-bearing `format_unsupported` forward hook, §9.7.4). Fourth member of the vault-singleton blob family with `vault-prefs` (MYP1) + `vault-search` (MYS1) + `myn-blob` (MY01); landed in lockstep with the `hkdf-sha256` `session-state-key` role + the `ctx-aead`/`aad-construction` `aad_workspace_blob` pins.
- `vectors/property-coercion/` — the Saved Views property value-typing / coercion ruleset (`markdown-dialect.md` §3.6.4, MYND-271). Each case is a one-key YAML 1.2 mapping `prop: <value>`; a conformant client parses it under the YAML 1.2 **core** schema and coerces the value to exactly one Myne type — `string` / `number` / `date` / `list` / `checkbox`. Deterministic cross-client behavior in the `password-policy` / `note-template` class (not a codec, not cryptographic). 31 cases via a 2-way `case_kind` (`scalar`: 26 — incl. the load-bearing tie-breaks `007`→string, `2026`→number, `2026-06`→string, `yes`/`on`/`True`→string, the quoting-demotes-checkbox/number-but-not-date asymmetry, and the NFD byte-transparency pin; `list`: 5 — flow/block, heterogeneous per-element coercion, empty, single). property-index is the **single type authority**; sibling features consume this typing. Hand-curated (no Rust generator — the coercion is TS-side and asserted against these vectors by the property-index Vitest runner).
- `vectors/markdown-comment/` — the Myne dialect `%%…%%` editor-only comment (`markdown-dialect.md` §3.1 block / §3.2 inline / §3.7.5 exclusion / §6 strip, MYND-315). Each case is a markdown source (hex); a conformant client recognizes terminated `%%…%%` comments (inline + multi-line block) and strips them on lossy projections (HTML / PDF / stripped-TXT / document statistics). Deterministic cross-client behavior in the `property-coercion` / `note-template` class (not a codec, not cryptographic). 14 cases via a 3-way `case_kind` (`recognize`: 7 — inline/block/multiline/empty/multiple/mixed; `inert`: 3 — the load-bearing **unterminated-`%%`-is-inert** tie-breaks + single-`%`; `exclusion`: 4 — `%%` inside fenced code / inline code / wikilink / inline-math is literal). Pins `expected_has_comment` + `expected_stripped_utf8_hex` (byte-range removal; `.md` is exempt, SYNC-2). Hand-curated (no Rust generator — the recognizer + strip are TS-side, asserted by the dialect Vitest runner).
- `vectors/vault-indices/` — `encode_vault_indices_blob_raw` + `decode_vault_indices_blob` `MYI1` `vault.indices` encrypted structured-metadata codec (MYND-271; design MYND-31, spec §9.8). Opaque inner plaintext (the desktop layer's serialized structured index — wikilink graph + tag index + title map + typed property records; no canonical-JSON, like vault-search/vault-workspace); KEK `derive_subkey(root_key, SubkeyRole::VaultIndicesKey)` (role `vault-indices-key`) and AAD `aad_vault_indices_blob` (role `vault-indices-blob`). **Distinct blob family from `vault-search`** (`MYI1` != `MYS1`, one magic + one KEK + one AAD role per family). 12 cases (the `vault-search`/`vault-workspace` shape): 3 encrypt + 1 decrypt_ok + 2 non-distinguishing decrypt_aead_fail (real-key fixtures) + 6 distinguishable parse_fail (incl. the load-bearing `format_unsupported` §9.8.4 forward hook). Landed in lockstep with the `hkdf-sha256` `vault-indices-key` role + the `ctx-aead`/`aad-construction` `aad_vault_indices_blob` builder.
- `vectors/vault-dictionary/` — `encode_dictionary_blob_raw` + `decode_dictionary_blob` `MYD1` `vault.dictionary` encrypted personal-dictionary codec (MYND-266, spec §9.13). Opaque inner plaintext (the desktop layer's serialized `{personal_words, ignore_words}` word sets — no canonical-JSON, like vault-search/workspace/indices); KEK `derive_subkey(root_key, SubkeyRole::PersonalDictKey)` (role `personal-dictionary-key`) and AAD `aad_personal_dictionary_blob` (role `personal-dictionary-blob` — a **distinct** load-bearing string). 12 cases: 3 encrypt + 1 decrypt_ok + 2 decrypt_aead_fail (non-distinguishing, §6.4) + 6 `parse_fail` (one per distinguishable pre-decryption structural reject incl. the load-bearing `format_unsupported` forward hook, §9.13.4 — the Rust variant is `DictFormatUnsupported`, kept as the `format_unsupported` vector string for cross-family consistency). Sixth member of the vault-singleton blob family. Landed in lockstep with the `hkdf-sha256` `personal-dictionary-key` role + the `ctx-aead`/`aad-construction` `aad_personal_dictionary_blob` builder. Generator: `emit_vault_dictionary_vectors_for_paste` (`dictionary_blob.rs`).
- `vectors/vault-quickunlock/` — `encode_quick_unlock_blob_raw` + `decode_quick_unlock_blob` `MYQ1` `vault.quickunlock` device-local credential codec (MYND-329, spec §9.14). Wraps a **third copy of `root_key`** (the §3.1 "third copy") under a provider-supplied `wrap_key`; the codec is **provider-agnostic** and never runs Argon2id, so vectors supply `wrap_key` directly (the PIN→`wrap_key` derivation is pinned separately as the `argon2id` `quickunlock-pin` path). **No HKDF `SubkeyRole`** — a `root_key`-derived KEK would be circular (§3.4); only the AAD role `quick-unlock-blob`, with `provider_kind` (0x01 biometric / 0x02 PIN) AAD-bound. Fixed 64-byte ciphertext (no size cap → `CiphertextLenInvalid`, not `…Overflow`). 12 cases: 2 encrypt (biometric + PIN) + 1 decrypt_ok + 3 decrypt_aead_fail (tag-tamper + cross-vault + provider-flip — provider-flip is AEAD, not structural, because both 0x01/0x02 are registered) + 6 `parse_fail`. Landed in lockstep with the `ctx-aead`/`aad-construction` `aad_quick_unlock_blob` builder + the `argon2id` `quickunlock-pin` derivation path. Generator: `emit_quick_unlock_vectors_for_paste` (`quick_unlock_blob.rs`).
- `vectors/backup-container/` — the `MYB1` `.myne-backup` whole-vault backup container (MYND-254; codec landed MYND-217, spec §9.11): `write_container` + `read_and_verify_container` + the §9.11.3 closed-allowlist classifier. **First family whose reference implementation, generator and runner live in the main repo's `src-tauri/` (package `myne-desktop`), not `protocol/`** — the codec is std/fs/streaming-bound and cannot enter the no_std protocol crate. Credential-free UNKEYED integrity layer (per-member SHA-256 over `path_len_be16||path||member_bytes` — corruption detection, never tamper resistance); member content opaque, vault.meta structural parse = restore-engine stage 5, deliberately unvectored. 32 cases via a 4-way `case_kind`: 2 `encode` (unsorted input pins writer-sorts canonical raw-byte order) + 2 `verify_ok` + 14 `verify_fail` (all 13 vectorable error variants + the MYB5 v005-reservation magic; the two ceiling cases pin the §9.11.6 stage order) + 14 `classify_path` (6 accepts + 8 load-bearing rejects incl. the reserved-out `vault.search`/`vault.workspace`).

### Planned

**None.** Every previously-planned category (`bip39-recovery`, `aad-construction`) has landed alongside `password-policy`. Future major-version (`v005`) vectors land under `vectors/<cat>/v005/` when that epoch opens.

## JSON schema convention

Every vector file:
- Hex-encoded inputs (`key`, `nonce`, `aad`, `salt`, etc.) — lowercase, no `0x` prefix
- Hex-encoded expected outputs
- `spec_section` reference (e.g., `"protocol.md §5.1.1"`)
- `vector_id` (stable string, never reused)
- `protocol_version: "v0.1"` — pinned per vector so v005+ vectors can coexist without ambiguity

Field naming convention applied across every `cases.json`:
- Identity: `vector_id` (string, format `<category>/<version>/<descriptive-slug>`)
- Discriminator: `case_kind` (optional, present only when a category has more than one case shape)
- Inputs: `input_<semantic>_<encoding>` (e.g., `input_root_key_hex`, `input_plaintext_utf8_hex`)
- Expected outputs: `expected_<semantic>_<encoding>` (e.g., `expected_subkey_hex`, `expected_prefs_blob_hex`)
- Failure-mode expectation: `expected_kind` (string, e.g., `aead_fail`)
- Informational / intermediate: `informational_<semantic>[_<encoding>]` (suffix optional for structured objects)

The canonical machine-readable contract lives at `schema/v0.1.json` (top-level shape) and `schema/v0.1/<category>.json` (per-category case shape). The root schema dispatches via `oneOf` keyed on the `schema` field.

## v005 reservation

Future major-version vectors will live under `vectors/<cat>/v005/` subdirectories alongside v0.1 (`vectors/<cat>/v0.1/`), validated by a separate `schema/v005.json`. v005 magic bytes: `MY05` / `MYV5`; HKDF prefix: `"myne.v005 "`. v0.1 and v005 are **forward-incompatible by design** — they share this repo for distribution convenience, not for coexistence within a single client.

## Cross-implementation discipline

Independent ports (Phase 2 Go server, Phase 3 web/mobile clients) verify primitive-level compatibility by running these vectors through their language's runner. **A failed vector in any port is a stop-the-world bug** — either the port has drifted or the spec is wrong. Don't regenerate vectors to "fix" a failing port without reading why.

## Common commands

```bash
# Validate every landed cases.json against its canonical sub-schema:
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
  --spec=draft2020

# Inspect a single case via jq:
jq '.cases[0]' vectors/hkdf-sha256/v0.1/cases.json
```

## Regeneration

Each category's vector set is generated by a corresponding `#[ignore]`-gated test in the main repo (`protocol/`, or `src-tauri/` for `backup-container`), so regenerating vectors is always a deliberate act:

| Category | Generator | Output |
|---|---|---|
| `vault-prefs` | `cargo test -p protocol --lib emit_prefs_vectors_for_paste -- --ignored` | `/tmp/myne_vault_prefs_vectors.txt` (paste-format) |
| `vault-prefs` (label + tolerance cases) | `cargo test -p protocol --lib emit_label_and_tolerance_prefs_vectors -- --ignored` | `/tmp/myne_vault_prefs_label_vectors.txt` (paste-format; MYND-205 label pair + §9.5.2 unknown-field tolerance case) |
| `vault-prefs` (last_backup_at cases) | `cargo test -p protocol --lib emit_last_backup_at_prefs_vectors -- --ignored` | `/tmp/myne_vault_prefs_last_backup_vectors.txt` (paste-format; MYND-58 epoch-seconds pair) |
| `vault-prefs` (parse_fail cases) | `cargo test -p protocol --lib emit_prefs_parse_fail_vectors -- --ignored` | `/tmp/myne_vault_prefs_parse_fail_vectors.json` (JSON array fragment to splice; MYND-227, 7 cases) |
| `vault_meta_rewrite` | `cargo test -p protocol --lib emit_vector_hex_for_paste -- --ignored` | `/tmp/myne_vault_meta_rewrite_vector.txt` (paste-format; 3 blocks, **~6 s** — two canonical Argon2id derivations for the `canonical-kdf-params` block + its self-validation) |
| `hkdf-sha256` | `cargo test -p protocol --lib emit_subkey_vectors_for_paste -- --ignored` | `/tmp/myne_hkdf_sha256_vectors.json` (full JSON, copy verbatim) |
| `vault-meta` | `cargo test -p protocol --lib emit_vault_meta_vectors_for_paste -- --ignored` | `/tmp/myne_vault_meta_vectors.json` (full JSON) |
| `vault-meta` (parse_fail cases) | `cargo test -p protocol --lib emit_vault_meta_parse_fail_vectors -- --ignored` | `/tmp/myne_vault_meta_parse_fail_vectors.json` (JSON array fragment to splice; MYND-227, 4 cases) |
| `account-number` | `cargo test -p protocol --lib emit_account_number_vectors_for_paste -- --ignored` | `/tmp/myne_account_number_vectors.json` (full JSON) |
| `myn-blob` | `cargo test -p protocol --lib emit_myn_blob_vectors_for_paste -- --ignored` | `/tmp/myne_myn_blob_vectors.json` (full JSON; ~425 KB due to 100-KB plaintext case) |
| `myn-blob` (parse_fail cases) | `cargo test -p protocol --lib emit_myn_blob_parse_fail_vectors -- --ignored` | `/tmp/myne_myn_blob_parse_fail_vectors.json` (JSON array fragment to splice; MYND-227, 8 parse_fail + 1 aead-fail) |
| `ctx-aead` | `cargo test -p protocol --lib emit_ctx_aead_vectors_for_paste -- --ignored` | `/tmp/myne_ctx_aead_vectors.json` (full JSON) |
| `argon2id` | `cargo test -p protocol --lib emit_argon2id_wrap_vectors_for_paste -- --ignored` | `/tmp/myne_argon2id_vectors.json` (full JSON; 7 cases incl. the MYND-329 `quickunlock-pin` path; **~21 s runtime** at canonical params) |
| `vault-search` | `cargo test -p protocol --lib emit_vault_search_vectors_for_paste -- --ignored` | `/tmp/myne_vault_search_vectors.json` (full JSON) |
| `password-policy` | `cargo test -p protocol --lib emit_password_policy_vectors_for_paste -- --ignored` | `/tmp/myne_password_policy_vectors.json` (full JSON; generator in `protocol/src/password_policy.rs`) |
| `bip39-recovery` | `cargo test -p protocol --lib emit_bip39_recovery_vectors_for_paste -- --ignored` | `/tmp/myne_bip39_recovery_vectors.json` (full JSON; **~9 s runtime**; generator in `protocol/src/kdf.rs`, asserts all variants derive the same key) |
| `aad-construction` | `cargo test -p protocol --lib emit_aad_construction_vectors_for_paste -- --ignored` | `/tmp/myne_aad_construction_vectors.json` (full JSON; generator in `protocol/src/aead.rs`) |
| `damm32-table` | `cargo test -p protocol --lib emit_damm32_table_vectors_for_paste -- --ignored` | `/tmp/myne_damm32_table_vectors.json` (full JSON; generator in `protocol/src/account_number.rs`, cross-checks the table vs an independent srobo/damm32 transcription at mint) |
| `attachment-blob` | `cargo test -p protocol --lib emit_attachment_blob_vectors_for_paste -- --ignored` | `/tmp/myne_attachment_blob_vectors.json` (full JSON) |
| `attachment-blob` (parse_fail cases) | `cargo test -p protocol --lib emit_attachment_blob_parse_fail_vectors -- --ignored` | `/tmp/myne_attachment_blob_parse_fail_vectors.json` (JSON array fragment to splice; MYND-227, 8 parse_fail + 1 aead-fail) |
| `attachments-catalog` | `cargo test -p protocol --lib emit_attachments_catalog_vectors_for_paste -- --ignored` | `/tmp/myne_attachments_catalog_vectors.json` (full JSON) |
| `attachments-catalog` (parse_fail cases) | `cargo test -p protocol --lib emit_catalog_parse_fail_vectors -- --ignored` | `/tmp/myne_attachments_catalog_parse_fail_vectors.json` (JSON array fragment to splice; MYND-227, 7 cases) |
| `note-template` | `cargo test -p protocol --lib emit_note_template_vectors_for_paste -- --ignored` | `/tmp/myne_note_template_vectors.json` (full JSON; generator in `protocol/src/template.rs`, self-asserts every expectation at mint) |
| `vault-workspace` | `cargo test -p protocol --lib emit_vault_workspace_vectors_for_paste -- --ignored` | `/tmp/myne_vault_workspace_vectors.json` (full JSON; generator in `protocol/src/workspace_blob.rs`) |
| `vault-indices` | `cargo test -p protocol --lib emit_vault_indices_vectors_for_paste -- --ignored` | `/tmp/myne_vault_indices_vectors.json` (full JSON; generator in `protocol/src/vault_indices_blob.rs`) |
| `vault-dictionary` | `cargo test -p protocol --lib emit_vault_dictionary_vectors_for_paste -- --ignored` | `/tmp/myne_vault_dictionary_vectors.json` (full JSON; generator in `protocol/src/dictionary_blob.rs`) |
| `vault-quickunlock` | `cargo test -p protocol --lib emit_quick_unlock_vectors_for_paste -- --ignored` | `/tmp/myne_vault_quickunlock_vectors.json` (full JSON; generator in `protocol/src/quick_unlock_blob.rs`) |
| `attachment-text` | `cargo test -p protocol --lib emit_attachment_text_vectors_for_paste -- --ignored` | `/tmp/myne_attachment_text_vectors.json` (full JSON; generator in `protocol/src/attachment_text_blob.rs`) |
| `backup-container` | `cargo test -p myne-desktop --lib emit_backup_container_vectors_for_paste -- --ignored` | `/tmp/myne_backup_container_vectors.json` (full JSON; generator in `src-tauri/src/backup_container.rs` — note the package: `-p myne-desktop`, NOT `-p protocol`; self-asserts every expectation at mint) |

`phrase_normalize` vectors are hand-curated against the protocol §7.4 pipeline; no generator. `property-coercion` is likewise hand-curated against `markdown-dialect.md` §3.6.4 — there is no Rust generator because the coercion lives TS-side (the property-index sub-feature, MYND-271); the property-index Vitest runner asserts the live TS coercion against these vectors. `markdown-comment` is hand-curated against `markdown-dialect.md` §3.1/§3.2/§3.7.5/§6 — no Rust generator (the `%%` recognizer + strip live TS-side in `src/dialect/lezer/comment.ts` + `src/dialect/stripComments.ts`); the dialect Vitest runner (`tests/dialect/markdownComment.test.ts`) hex-decodes each case, runs the live recognizer + strip, and asserts recognition + the strip bytes. The newer (Lane D) generators all emit ready-to-copy JSON; the two oldest (`vault-prefs`, `vault_meta_rewrite`) still emit paste-format text — both patterns coexist.

## Gotchas

- **NFKD normalization order matters** for BIP-39 phrase vectors — apply NFKD before passing to Argon2id, never after. Vectors must include both NFC and NFKD-equivalent inputs to catch implementations that skip the normalization step.
- **Hex encoding is lowercase, no `0x` prefix.** A vector's `expected_<output>_hex` of `1a2b3c...` not `0x1A2B3C...`. Don't mix cases within a single vector file.
- **Vector IDs are immutable** — once a vector is shipped, its `vector_id` is part of the contract; downstream test runners may reference it. Renaming requires a major version bump.
- **Bit-identical means byte-identical** — there is no "approximately equal" for these vectors. Endianness, padding, and reserved bytes all matter. Cross-language tests should diff bytes, not parsed structures.
- **Adding a new category** requires landing a sub-schema file under `schema/v0.1/`, extending the root schema's `schema` enum AND `oneOf` branches in lockstep, and adding the corresponding `cases.json` + integration runner in the main repo.
