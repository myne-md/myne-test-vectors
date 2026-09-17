# `agent-memory-blob` vector schema

Pins the byte mapping of the `memories/<memory-id-hex>` per-entry agent-memory codec (`MYX1`, myne-md/myne#697): `encode_agent_memory_blob_raw` + `decode_agent_memory_blob` + `aad_agent_memory_blob` (Rust crate `protocol`, main `myne` repo). One KEK for the whole family (`agent_memory_key = DeriveSubkey(root_key, SubkeyRole::AgentMemoryKey)`); per-entry separation comes from the AAD binding (the `MYE1`/`chats-blob` archetype). Each entry holds the distilled memory text the agent wrote (via the `remember` verb, at most one write per run) or the user edited, plus the optional `source_conversation` link.

**Reference implementation:** `protocol/src/agent_memory_blob.rs` (the chats-blob shape).
**Spec:** `docs/spec/protocol.md` § 9.26 (`MYX1` envelope + entry inner shape + report-don't-empty + 256-KiB cap) + § 5.2 / § 9.26.3 `AAD_agent_memory_blob` formula (the AAD role `"agent-memory-blob"`, which also binds `memory_entry_id` (16 B) + `account_number_hash` (16 B), 62 B total) + § 3.4 registry entry `"agent-memory-key"` (the family KEK role).

## Canonical entry inner plaintext

The inner plaintext is the client layer's canonical entry JSON: an object with `text` (NFC-normalized string, at most 2000 Unicode scalar values after the host's write-time cap) and the optional `source_conversation` (32-hex conversation id, present only on a model-written entry). Key order is NOT significant; writers emit `text` first and parsers accept either order. Entries are **mutable whole values, never append-merged**: an entry is one text the agent rewrites or the user edits; causal ordering is the catalog's job (§ 9.26.2).

## The per-entry binding

- **KEK (HKDF) role** `"agent-memory-key"` (16 ASCII bytes) — one KEK for the whole family; per-entry safety comes from this AAD. Pinned independently in the `hkdf-sha256` category.
- **AAD role** `"agent-memory-blob"` (17 ASCII bytes, length-prefixed) — `AAD_agent_memory_blob = "myne.v0.1|" (10 B) || protocol_version 0x01 (1 B) || "agent-memory-blob" (19 B total with the 2-B BE length prefix) || memory_entry_id (16 B, also the on-disk filename hex) || account_number_hash (16 B) = 62 bytes`. Binding both ids prevents substituting one entry's blob for another within the same vault and porting a blob across vaults at the AEAD layer. Pinned by the `aad-construction` case (the segments list carries the per-segment breakdown, including the `memory_entry_id` label).

## The v0.2 side

`vectors/agent-memory-blob/v0.2/` pins the migrated envelope (`MYX5`, v0.2 KEK + v0.2 per-entry AAD): one `encrypt` case plus the `decode_fail_v0_1` cross-generation refusal. Unlike the singletons, the per-entry families carry the generation in the header version byte (`0x02`).

## File layout

```
vectors/agent-memory-blob/
├── schema.md              ← this file
├── v0.1/
│   └── cases.json         ← array of 13 cases (3 encrypt + 1 decrypt_ok + 3 decrypt_aead_fail + 6 parse_fail)
└── v0.2/
    └── cases.json         ← MYX5 envelope + cross-generation refusal
```

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/agent-memory-blob.json`](../../schema/v0.1/agent-memory-blob.json) (per-case shape with a 4-way `case_kind` discriminator). The sub-schema is authoritative — this document covers only the domain-specific semantics.
