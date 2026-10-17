# `agent-memory-catalog` vector schema

Pins the byte mapping of the `vault.agent-memory` conversations-catalog-shaped codec (`MYN1`, myne-md/myne#697): `encode_agent_memory_catalog_blob_raw` + `decode_agent_memory_catalog_blob` + `aad_agent_memory_catalog` (Rust crate `protocol`, main `myne` repo). `vault.agent-memory` is the vault-level singleton encrypted **agent-memory catalog** (`protocol.md` §9.25) that lives next to `vault.chats` / `vault.prefs` and maps memory-entry ids to their roster rows. Written only while the runtime beta opt-in `ai_enabled` is on (the same runtime beta opt-in gate the other AI-backed families follow); synced Class-1, backed up.

**Reference implementation:** `protocol/src/agent_memory_blob.rs` (the chats-catalog shape).
**Spec:** `docs/spec/protocol.md` § 9.25 (envelope + roster inner shape + report-don't-empty) + § 5.2 (AAD formula, role `"agent-memory-catalog"`, 49-byte AAD) + § 3.4 registry entries `"agent-memory-catalog-key"` (the KEK role) and `"agent-memory-catalog-sync-id"` (the Phase-2 sync blob-id role).

## Canonical roster inner plaintext (like `chats-catalog` / `attachments-catalog`)

The inner plaintext is the client layer's canonical roster JSON: an object mapping memory-id hex → `{created_at, deleted_at?, text, updated_at, version?}`. Canonical form fixed by the writer:

- Top-level keys (memory ids) in lexicographic order; per-entry keys in alphabetical order `created_at` < `deleted_at` < `text` < `updated_at` < `version`, with `version: 0` omitted and `deleted_at` present only on tombstones.
- No insignificant whitespace; empty roster = `{}` (2 bytes, `7b7d`).

There is no forward-compat unknown-field tolerance case: the catalog is **load-bearing user content**, so an unreadable catalog is REPORTED (§ 9.25.3 report-don't-empty, never collapsed to "no memories"), and tolerance of unknown fields would silently drop roster rows a newer client wrote.

## Two distinct load-bearing role strings, plus the sync identifier

- **KEK (HKDF) role** `"agent-memory-catalog-key"` (25 ASCII bytes) — `catalog_key = DeriveSubkey(root_key, "agent-memory-catalog-key")` per § 3.2 + § 3.4. Pinned independently in the `hkdf-sha256` category.
- **Sync-id (HKDF) role** `"agent-memory-catalog-sync-id"` (29 ASCII bytes) — the Phase-2 sync blob-id of `vault.agent-memory` (first 16 bytes of the output), an identifier derivation like `chats-catalog-sync-id`, NOT a KEK. Pinned independently in the `hkdf-sha256` category.
- **AAD role** `"agent-memory-catalog"` (20 ASCII bytes, length-prefixed) — the role inside `aad_agent_memory_catalog`: `"myne.v0.1|"` (10 B) + protocol_version `0x01` (1 B) + 2-B BE length `0x0014` + role (20 B) + `account_number_hash` (16 B) = 49 bytes. Pinned by the `aad-construction` case.

## The v0.2 side

`vectors/agent-memory-catalog/v0.2/` pins the migrated envelope (`MYN5`, v0.2 KEK via `derive_subkey_versioned` + v0.2 AAD): one `encrypt` case plus the `decode_fail_v0_1` cross-generation refusal. The header version byte stays `0x01` (the envelope version, not the generation), like the chats-catalog singleton family.

## File layout

```
vectors/agent-memory-catalog/
├── schema.md              ← this file
├── v0.1/
│   └── cases.json         ← array of 13 cases (3 encrypt + 2 decrypt_ok + 2 decrypt_aead_fail + 6 parse_fail)
└── v0.2/
    └── cases.json         ← MYN5 envelope + cross-generation refusal
```

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/agent-memory-catalog.json`](../../schema/v0.1/agent-memory-catalog.json) (per-case shape with a 4-way `case_kind` discriminator). The sub-schema is authoritative — this document covers only the domain-specific semantics.
