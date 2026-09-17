# `agent-personality` vector schema

Pins the byte mapping of the `vault.personality` singleton codec (`MYZ1`, myne-md/myne#697): `encode_agent_personality_blob_raw` + `decode_agent_personality_blob` + `aad_agent_personality_blob` (Rust crate `protocol`, main `myne` repo). `vault.personality` carries the user's standing voice directives for the AI (`protocol.md` §9.27). **User-authored only**: written from Settings, never by the agent (no verb writes it, `decisions.md` 2026-09-10). Written only while `ai_enabled` is on; synced as whole-value LWW (`sync-protocol.md` §10.2.7, the themes-palette rule); backed up.

**Reference implementation:** `protocol/src/agent_personality_blob.rs` (the prefs/themes singleton shape).
**Spec:** `docs/spec/protocol.md` § 9.27 (`MYZ1` envelope + directives inner shape + report-don't-empty + 256-KiB cap) + § 5.2 (AAD formula, role `"personality-blob"`, 45-byte AAD) + § 3.4 registry entries `"agent-personality-key"` (the KEK role) and `"agent-personality-sync-id"` (the Phase-2 sync blob-id role).

## Canonical personality inner plaintext

The inner plaintext is the client layer's canonical personality JSON: an object with one key, `directives` — an array of at most 32 directive objects, each carrying `text` (NFC-normalized string, at most 2000 Unicode scalar values) and `updated_at` (epoch-ms integer), per-element keys emitted in lexicographic order (`text` < `updated_at`). **Array order is directive precedence, preserved verbatim — never sorted by any id** (there is no id): the agent reads them in order and first-match-wins is the natural reading of an ordered rule list. The object is omitted-when-empty at the file level: a vault whose user never set a personality has no `vault.personality` file at all, and clearing every directive removes the file.

## Two distinct load-bearing role strings, plus the sync identifier

- **KEK (HKDF) role** `"agent-personality-key"` (22 ASCII bytes) — `personality_key = DeriveSubkey(root_key, "agent-personality-key")` per § 3.2 + § 3.4. Pinned independently in the `hkdf-sha256` category.
- **Sync-id (HKDF) role** `"agent-personality-sync-id"` (26 ASCII bytes) — the Phase-2 sync blob-id of `vault.personality` (first 16 bytes of the output), an identifier derivation like `themes-sync-id`, NOT a KEK. Pinned independently in the `hkdf-sha256` category.
- **AAD role** `"personality-blob"` (16 ASCII bytes, length-prefixed) — the role inside `aad_agent_personality_blob`: `"myne.v0.1|"` (10 B) + protocol_version `0x01` (1 B) + 2-B BE length `0x0010` + role (16 B) + `account_number_hash` (16 B) = 45 bytes. Pinned by the `aad-construction` case.

## The v0.2 side

`vectors/agent-personality/v0.2/` pins the migrated envelope (`MYZ5`, v0.2 KEK via `derive_subkey_versioned` + v0.2 AAD): one `encrypt` case plus the `decode_fail_v0_1` cross-generation refusal. The header version byte stays `0x01` (the envelope version, not the generation).

## File layout

```
vectors/agent-personality/
├── schema.md              ← this file
├── v0.1/
│   └── cases.json         ← array of 10 cases (1 encrypt + 1 decrypt_ok + 2 decrypt_aead_fail + 6 parse_fail)
└── v0.2/
    └── cases.json         ← MYZ5 envelope + cross-generation refusal
```

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/agent-personality.json`](../../schema/v0.1/agent-personality.json) (per-case shape with a 4-way `case_kind` discriminator). The sub-schema is authoritative — this document covers only the domain-specific semantics.
