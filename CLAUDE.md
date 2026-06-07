# myne-test-vectors — JSON test vectors for Myne v0.1 protocol

> Companion repository to [`myne`](https://github.com/myne-md/myne). Public, **CC0** (public domain) — vectors must remain maximally reusable for independent implementations in any language.

## Purpose

Language-agnostic JSON test vectors for the Myne cryptographic protocol. Any independent implementation (Rust, Go, Swift, Kotlin, JavaScript, etc.) MUST produce **bit-identical outputs** from these inputs. A reviewer or contributor verifying a port should not need to read the reference Rust implementation; running their language's test runner against this repository should be enough.

## Status

**Phase 1, fourteen v0.1 categories landed.** The `schema/v0.1.json` root schema + per-category sub-schemas under `schema/v0.1/` are the authoritative machine-readable contract. Landed: `phrase_normalize`, `vault-prefs`, `vault_meta_rewrite`, `hkdf-sha256`, `vault-meta`, `account-number`, `myn-blob`, `ctx-aead`, `argon2id`, `vault-search`, `password-policy`, `bip39-recovery`, `aad-construction`, `damm32-table`. `password-policy` / `bip39-recovery` / `aad-construction` closed the implementation-vs-vectors gaps for surfaces unpinned by cross-impl vectors; `damm32-table` (MYND-74) pins the frozen Damm-32 quasigroup table itself as cross-impl ground truth alongside the existing `account-number` parse-behavior vectors. **No categories remain planned.** CI integration is currently deferred — schema validation and byte-level integration run as developer-side steps. See main repo `agent tooling/features/test-vector-schema/` (the canonical schema mechanism + first 4 categories), `agent tooling/features/lane-d-completion/` (the next 5), and `agent tooling/features/damm32-table-freeze/` (`damm32-table`) for the landing plans.

## Vector categories

### Landed (v0.1)

- `vectors/phrase_normalize/` — `protocol::phrase::normalize_user_phrase` NFKD + ASCII-lowercase + whitespace pipeline (spec §7.4).
- `vectors/vault-prefs/` — `MYP1` encrypted-preferences blob codec; encrypt + decrypt + non-distinguishing AEAD-failure case_kinds (spec §9.5 + §6.4).
- `vectors/vault_meta_rewrite/` — `MYV1` password-side rewrite producing the 345-byte vault.meta (spec §9.3 + recovery-flow §4.2/§4.4).
- `vectors/hkdf-sha256/` — Subkey derivations through `derive_subkey` exercising every registered `SubkeyRole` (`itemsKey-wrap`, `prefs-key`, `search-index-key`) with `myne.v0.1 ` info-prefix (spec §3.2 + §3.4).
- `vectors/vault-meta/` — `serialize_vault_meta` + `parse_vault_meta` create-time MYV1 encoder/decoder pair; round-trip + structural-parse-reject (magic / reserved tamper) (spec §9.3).
- `vectors/account-number/` — `parse_account_number` Crockford-base32 + Damm-32 + liberal-parsing normalization; parse permutations + strict-reject + Damm-32 substitution/transposition catches (spec §5 + decisions.md 2026-04-28 polynomial freeze).
- `vectors/myn-blob/` — `encode_new_note_blob_raw` + `encode_update_blob_raw` + `decode_note` MY01 per-note blob. 9 cases: 3 encrypt (10 B / 1 KB / 100 KB) + 3 `encrypt_update` (§6.3 note-update: same-size/grow/shrink, each pinning the verbatim 112-byte `header_prefix` splice — `enc_item_key` never re-wraps, §3.3) + 2 `decrypt_ok` (round-trip + post-update) + 1 ciphertext-tamper reject (spec §9.2 + §6.3 + §6.4); `content_type` hardcoded `text/markdown`.
- `vectors/ctx-aead/` — `aead_seal_ctx` + `aead_open_ctx` AEAD primitive + every pub `aad_*` builder's canonical bytes (4-way `case_kind` including `aad_canonical_bytes`); 7 builders pinned individually (incl. `aad_search_index_blob`, MYND-29) (spec §5.1 + §5.2 + §6.4). Tracked in earlier planned list as `xchacha20-poly1305-ctx`; renamed `ctx-aead` on landing.
- `vectors/argon2id/` — `derive_password_wrap_key` + `derive_recovery_wrap_key_from_normalized` at canonical Phase 1 Argon2id params (m=256 MiB, t=3, p=2, v=0x13); password + recovery paths × {short, long, unicode} (spec §3 + decisions.md 2026-04-26 KDF calibration). **~18s runtime cost** at canonical params; vectors are default-on (not `#[ignore]`-gated).
- `vectors/vault-search/` — `encode_search_index_blob_raw` + `decode_search_index_blob` `MYS1` `vault.search` encrypted search-index codec (MYND-29 / D8, spec §9.6). Opaque inner plaintext (no canonical-JSON, unlike vault-prefs); KEK `derive_subkey(root_key, SubkeyRole::SearchIndexKey)` (role `search-index-key`) and AAD `aad_search_index_blob` (role `search-index-blob` — a **distinct** load-bearing string). 12 cases: 3 encrypt + 1 decrypt_ok + 2 decrypt_aead_fail (non-distinguishing, §6.4) + 6 `parse_fail` (distinguishable pre-decryption structural rejects). Third member of the singleton-blob family with `vault-prefs` (MYP1) + `myn-blob` (MY01).
- `vectors/password-policy/` — `validate_min_length` master-password floor (`MIN_PASSWORD_LEN = 12`); **boolean accept/reject conformance** (not byte-output). Counts USVs (`str::chars().count()`), NOT UTF-8 bytes (decisions.md 2026-05-06). 7 cases: ASCII boundary (11 reject / 12 / 13 accept) + empty + the USV-vs-byte discriminators (`á`×12 accept / `á`×11 reject / mixed-script). Single case shape, `expected_kind` discriminator (no `case_kind`).
- `vectors/bip39-recovery/` — end-to-end recovery composition `normalize_user_phrase` (§7.4) → `derive_recovery_wrap_key_from_normalized` (§3.1): raw 24-word phrase NFKD-normalized then fed **directly** to Argon2id (NOT BIP-39's PBKDF2 step). 3 cases type the canonical all-zero phrase three ways (canonical / mixed-case+whitespace / NFKD full-width+exotic-space); all derive the **identical** wrap key, pinning NFKD-before-Argon2id. **~3s/case, default-on** like `argon2id`.
- `vectors/aad-construction/` — per-call-site AAD framing decomposition for all 7 pub `aad_*` builders. Pins the full AAD bytes (== `ctx-aead`'s `aad_canonical_bytes` pins) **plus** the segment breakdown (`aad_prefix` / `protocol_version` / `role_length_be16` / `role_label` / bound params) with offsets — a decomposition layer atop `ctx-aead`, not a duplicate. 7 cases (one per builder); harness asserts full builder coverage.
- `vectors/damm32-table/` — the frozen Damm-32 order-32 quasigroup table (`account-number.md` §4.2) as cross-impl ground truth (MYND-74). 4 cases via a 2-way `case_kind`: 1 `table` (the full 1024-byte row-major table as hex — `table[interim][symbol]`, GF(2⁵) with x∗y=α·(x⊕y), poly `0b100101=37`, Crockford index order) + 3 `checksum` (left-fold-from-0 orientation: single-step `[1]→2`, baseline `0..18→26`, all-high `31×19`). Reference: `protocol::damm32_table()` / `damm32_step`. Complements `account-number` (which pins `parse_account_number` behavior); this pins the table + raw index checksum beneath it.

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
  --spec=draft2020

# Inspect a single case via jq:
jq '.cases[0]' vectors/hkdf-sha256/v0.1/cases.json
```

## Regeneration

Each category's vector set is generated by a corresponding `#[ignore]`-gated test in the main repo (`protocol/`), so regenerating vectors is always a deliberate act:

| Category | Generator | Output |
|---|---|---|
| `vault-prefs` | `cargo test -p protocol --lib emit_prefs_vectors_for_paste -- --ignored` | `/tmp/myne_vault_prefs_vectors.txt` (paste-format) |
| `vault_meta_rewrite` | `cargo test -p protocol --lib emit_vector_hex_for_paste -- --ignored` | `/tmp/myne_vault_meta_rewrite_vector.txt` (paste-format) |
| `hkdf-sha256` | `cargo test -p protocol --lib emit_subkey_vectors_for_paste -- --ignored` | `/tmp/myne_hkdf_sha256_vectors.json` (full JSON, copy verbatim) |
| `vault-meta` | `cargo test -p protocol --lib emit_vault_meta_vectors_for_paste -- --ignored` | `/tmp/myne_vault_meta_vectors.json` (full JSON) |
| `account-number` | `cargo test -p protocol --lib emit_account_number_vectors_for_paste -- --ignored` | `/tmp/myne_account_number_vectors.json` (full JSON) |
| `myn-blob` | `cargo test -p protocol --lib emit_myn_blob_vectors_for_paste -- --ignored` | `/tmp/myne_myn_blob_vectors.json` (full JSON; ~425 KB due to 100-KB plaintext case) |
| `ctx-aead` | `cargo test -p protocol --lib emit_ctx_aead_vectors_for_paste -- --ignored` | `/tmp/myne_ctx_aead_vectors.json` (full JSON) |
| `argon2id` | `cargo test -p protocol --lib emit_argon2id_wrap_vectors_for_paste -- --ignored` | `/tmp/myne_argon2id_vectors.json` (full JSON; **~18 s runtime** at canonical params) |
| `vault-search` | `cargo test -p protocol --lib emit_vault_search_vectors_for_paste -- --ignored` | `/tmp/myne_vault_search_vectors.json` (full JSON) |
| `password-policy` | `cargo test -p protocol --lib emit_password_policy_vectors_for_paste -- --ignored` | `/tmp/myne_password_policy_vectors.json` (full JSON; generator in `protocol/src/password_policy.rs`) |
| `bip39-recovery` | `cargo test -p protocol --lib emit_bip39_recovery_vectors_for_paste -- --ignored` | `/tmp/myne_bip39_recovery_vectors.json` (full JSON; **~9 s runtime**; generator in `protocol/src/kdf.rs`, asserts all variants derive the same key) |
| `aad-construction` | `cargo test -p protocol --lib emit_aad_construction_vectors_for_paste -- --ignored` | `/tmp/myne_aad_construction_vectors.json` (full JSON; generator in `protocol/src/aead.rs`) |
| `damm32-table` | `cargo test -p protocol --lib emit_damm32_table_vectors_for_paste -- --ignored` | `/tmp/myne_damm32_table_vectors.json` (full JSON; generator in `protocol/src/account_number.rs`, cross-checks the table vs an independent srobo/damm32 transcription at mint) |

`phrase_normalize` vectors are hand-curated against the protocol §7.4 pipeline; no generator. The newer (Lane D) generators all emit ready-to-copy JSON; the two oldest (`vault-prefs`, `vault_meta_rewrite`) still emit paste-format text — both patterns coexist.

## Gotchas

- **NFKD normalization order matters** for BIP-39 phrase vectors — apply NFKD before passing to Argon2id, never after. Vectors must include both NFC and NFKD-equivalent inputs to catch implementations that skip the normalization step.
- **Hex encoding is lowercase, no `0x` prefix.** A vector's `expected_<output>_hex` of `1a2b3c...` not `0x1A2B3C...`. Don't mix cases within a single vector file.
- **Vector IDs are immutable** — once a vector is shipped, its `vector_id` is part of the contract; downstream test runners may reference it. Renaming requires a major version bump.
- **Bit-identical means byte-identical** — there is no "approximately equal" for these vectors. Endianness, padding, and reserved bytes all matter. Cross-language tests should diff bytes, not parsed structures.
- **Adding a new category** requires landing a sub-schema file under `schema/v0.1/`, extending the root schema's `schema` enum AND `oneOf` branches in lockstep, and adding the corresponding `cases.json` + integration runner in the main repo.
