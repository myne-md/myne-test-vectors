# `chats-merge` vector schema

Pins the conversation merge rules in `myne-core` — `merge_chats_catalog` + `merge_chats_turns` (main `myne` repo, `core/src/plugins/sync/conflict.rs`, myne-md/myne#675). Pure logic over decoded plaintexts: no keys, no AEAD, no envelopes. This is the first vector category whose reference implementation lives outside the `protocol` crate — the merges are sync policy, not cryptographic framing, so they are pinned here rather than beside the codecs.

**Reference implementation:** `core/src/plugins/sync/conflict.rs::{merge_chats_catalog, merge_chats_turns}`.
**Spec:** `sync-protocol.md` § 11.4 (the per-key causal join + remove-wins rule) + § 10.2.5 (the conversation-file reconcile).

## The two merges

**Catalog** — the `MYM1` roster join, structurally the themes merge: a per-key join over the causal `version`, with a remove-wins total-order tie-break. A conversation deleted on one device while appended-to on another resolves remove-wins (the appended turns are discarded at merge). Disjoint keys union. An enrolling device with an empty roster adopts the server roster wholesale — tombstones included, so a conversation deleted before the enrollment does not resurrect there.

**Turns** — the `MYJ1` turn-list join over turn-id string arrays: the common prefix is kept verbatim once; both divergent suffixes are kept, ordered by the turn-id total order (lexicographic on the id string — the ids embed a client discriminator, so cross-client collisions cannot arise and the byte order is argument-independent). An id on both sides contributes once. Commutative, idempotent and associative by construction.

## File layout

```
vectors/chats-merge/
├── schema.md              ← this file
└── v0.1/
    └── cases.json         ← array of 12 logic cases (catalog + turns + turn content)
```

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/chats-merge.json`](../../schema/v0.1/chats-merge.json) (per-case shape with a 3-way `case_kind` discriminator). The sub-schema is authoritative — this document covers only the domain-specific semantics.

- **`catalog_merge`** — a two-sided roster join: `input_local` + `input_remote` rosters (conversation-id hex → `{created_at, deleted_at?, title, updated_at, version?}`), `expected_merged`. The runner asserts the join in BOTH argument orders, so commutativity is structural rather than restated in the expected value. Cases: the commutative join, higher-version-wins, tombstone-wins-over-append, enrollment-seed-intact.
- **`catalog_merge_assoc`** — a three-sided roster join: `input_a` + `input_b` + `input_c`, `expected_merged`. The runner asserts `(A+B)+C` equals `A+(B+C)` equals the expected value.
- **`turns_merge`** — a two-sided turn-id join: `input_local` + `input_remote` id arrays, `expected_merged`. Both argument orders asserted. Cases: commutative suffixes, prefix-verbatim-plus-both-suffixes, idempotent.
- **`turns_merge_content`** — two copies of ONE turn id, reconciled field by field: `input_local` + `input_remote` turn objects, `expected_merged`. Both argument orders asserted. The only field that can differ is the monotone decision marker `agentSummary.outcome`, whose decided set is closed (`applied`, `rejected`): an unknown value counts as undecided, the decided copy wins, two disagreeing decisions resolve by a fixed order on the value (`applied` before `rejected`), and the marker is grafted onto the local turn so a field only one side carries is preserved. Cases: undecided+applied, undecided+rejected, two-disagreeing, unknown-value. (myne-md/myne#894)
