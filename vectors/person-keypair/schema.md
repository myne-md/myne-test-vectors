# `person-keypair` — v0.2 person-keypair wrap-key derivation

Cross-implementation vectors for `protocol::derive_person_keypair_wrap_key_from_normalized`
+ `protocol::person_id_hash` under the v0.2 salt domain
`"myne.v0.2 argon2id-person-keypair|"` (protocol.md §3.1,
decisions/protocol/0592, myne-md/myne#698).

Each member holds one person keypair whose sealed file is unwrapped by a wrap key
derived from the person's OWN recovery phrase. The derivation is
vault-independent: the salt slot that carries `account_number_hash` on every
vault path carries the phrase-derived `person_id_hash` here
(`SHA-256("myne.person-id|" || phrase)[:16]`), never any vault's account hash.
The two cases use different normalized phrases under the same salt, so both the
Argon2id password and the bound person id change: two people never share a wrap
key for one salt, and the runner asserts the two derived keys differ.

Reference implementation: `protocol/src/kdf.rs`
(`derive_person_keypair_wrap_key_from_normalized`, `person_id_hash`).
Generator: `emit_person_keypair_vectors_for_paste` in that module
(`cargo test -p protocol --lib emit_person_keypair_vectors_for_paste -- --ignored`).

## Version note

This family lives under `v0.2/` (not `v0.1/`) because the salt domain is a v0.2
identity: the `"myne.v0.2 argon2id-person-keypair|"` prefix is a
major-version-event constant, and a v0.1 KDF path must never derive it. The
native + browser runners read `person-keypair/v0.2/cases.json`.

## File layout

```
vectors/person-keypair/
├── schema.md              ← this file
└── v0.2/
    └── cases.json         ← 2 cases (person-keypair-a + person-keypair-b)
```

## JSON shape

The machine-readable contract lives at [`schema/v0.2.json`](../../schema/v0.2.json)
(top-level shape) and [`schema/v0.2/person-keypair.json`](../../schema/v0.2/person-keypair.json)
(per-case shape). Note the **`v0.2` root schema**, not `v0.1`'s:
`protocol_version` is a `const` in each root, so the two are separate documents
and a v0.2 file is invalid against the v0.1 root by construction.

The sub-schema is authoritative; this document covers only the domain semantics.
