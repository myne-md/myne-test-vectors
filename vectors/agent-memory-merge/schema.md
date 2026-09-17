# `agent-memory-merge` vector schema

Pins the agent-memory catalog merge rules in `myne-core` — `merge_memory_catalog` (main `myne` repo, `core/src/plugins/sync/conflict.rs`, myne-md/myne#697). Pure logic over decoded plaintexts: no keys, no AEAD, no envelopes. Like `chats-merge`, this category's reference implementation lives outside the `protocol` crate — the merges are sync policy, not cryptographic framing.

**Reference implementation:** `core/src/plugins/sync/conflict.rs::merge_memory_catalog`.
**Spec:** `sync-protocol.md` § 10.2.6 (the agent-memory pair reconcile) + § 11.4 (the per-key causal join + remove-wins rule).

## The merge

**Catalog** — the `MYN1` roster join, structurally the chats/themes merge: a per-key join over the causal `version`, with a remove-wins total-order tie-break. An entry deleted on one device while edited on another resolves remove-wins (the concurrent edit is discarded; the cost is bounded by the one-write-per-run rule, `protocol.md` §9.26.2, rather than by a modal). Disjoint keys union. An enrolling device with an empty roster adopts the server roster wholesale — tombstones included, so an entry deleted before the enrollment does not resurrect there.

Unlike the conversation-file turn list, there is **no per-file merge**: an entry is one mutable text, not an append list, so the per-entry reconcile adopts the catalog-winner's bytes (the conversation-file cadence: content-addressed by id, the catalog owns causal ordering).

## File layout

```
vectors/agent-memory-merge/
├── schema.md              ← this file
└── v0.1/
    └── cases.json         ← array of 3 logic cases (2 catalog_merge + 1 catalog_merge_assoc)
```

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/agent-memory-merge.json`](../../schema/v0.1/agent-memory-merge.json) (per-case shape with a 2-way `case_kind` discriminator). The sub-schema is authoritative — this document covers only the domain-specific semantics.

- **`catalog_merge`** — a two-sided roster join: `input_local` + `input_remote` rosters (memory-id hex → `{created_at, deleted_at?, text, updated_at, version?}`), `expected_merged`. The runner asserts the join in BOTH argument orders, so commutativity is structural rather than restated in the expected value.
- **`catalog_merge_assoc`** — a three-sided roster join: `input_a` + `input_b` + `input_c`, `expected_merged`. The runner asserts `(A+B)+C` equals `A+(B+C)` equals the expected value.
