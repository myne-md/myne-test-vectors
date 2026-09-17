# `person-identity` — v0.2 canonical fingerprint + hybrid-wrap combiner

Cross-implementation vectors for the two deterministic person-path byte
constructions (protocol.md §3.1, decisions/protocol/0592, myne-md/myne#698):

- `compute_fingerprint(kem_ek, ed_pk, mldsa_vk)` =
  `SHA-256("myne.person-fingerprint v1|" || kem_ek || ed_pk || mldsa_vk)`,
  the canonical fingerprint two members compare (SAS) before trusting an
  invite. The domain prefix is load-bearing: the bare SHA-256 of the
  concatenation differs, and the runner asserts it does.
- `hybrid_wrap_key(classical_ss, pq_ss)` =
  `HKDF-SHA-256-Extract("myne.person-wrap-ss v1|", classical_ss || pq_ss)`,
  the combiner that folds the classical and post-quantum shared secrets into
  the key that wraps a per-recipient itemsKey. Argument order is fixed
  (classical first); a bare-hash combiner produces a different, wrong key.

A port that frames either differently derives a different fingerprint (so SAS
never matches) or a different wrap key (so no invite opens).

Reference implementation: `protocol/src/person_key.rs`
(`compute_fingerprint`, `hybrid_wrap_key`).
Generator: `emit_person_identity_vectors_for_paste` in that module
(`cargo test -p protocol --lib emit_person_identity_vectors_for_paste -- --ignored`).

## Version note

This family lives under `v0.2/` (not `v0.1/`) because both domain prefixes are
v0.2 identities. The native + browser runners read
`person-identity/v0.2/cases.json`.

## File layout

```
vectors/person-identity/
├── schema.md              ← this file
└── v0.2/
    └── cases.json         ← 2 cases (fingerprint + hybrid-wrap-key)
```

## JSON shape

The machine-readable contract lives at [`schema/v0.2.json`](../../schema/v0.2.json)
(top-level shape) and [`schema/v0.2/person-identity.json`](../../schema/v0.2/person-identity.json)
(per-case shape, 2-way `case_kind` discriminator). Note the **`v0.2` root
schema**, not `v0.1`'s: `protocol_version` is a `const` in each root, so the two
are separate documents and a v0.2 file is invalid against the v0.1 root by
construction.

The sub-schema is authoritative; this document covers only the domain semantics.

## Case kinds

| `case_kind` | What it pins |
|---|---|
| `fingerprint` | canonical fingerprint over three fixed public halves |
| `hybrid-wrap-key` | hybrid combiner over fixed classical + PQ shared secrets |
