# `person-account-number` — v0.2 person-derived account number

Cross-implementation vectors for `protocol::account_number_from_person_id`
+ `protocol::person_id_hash` (`docs/spec/account-number.md` §3.1,
`protocol.md` §3.1, myne-md/myne#911).

One number per PERSON rather than per vault. The 20-character canonical form is
a display encoding of the **leading 95 bits** of the 16-byte person id: the top
95 bits of the big-endian value (`u128 >> 33`, so the low 33 bits are discarded),
grouped five bits at a time most-significant-first through the Crockford
alphabet, plus the Damm-32 check character over those nineteen indices.

**Nothing else about the scheme moves.** The `MY-` prefix, the alphabet, the
95-bit width, the check character, the 4-character grouping and the
`account_number_hash` rule are the same values `account-number/v0.1` pins for a
generated number, so one parser validates both and the frozen table in
`account-number.md` §4.2 applies to both. What changes is only where the bits
come from.

**Why the top bits, and why that is frozen.** A person id is
`SHA-256("myne.person-id|" || phrase)[:16]`, so every 95-bit window of it is
uniformly random and the choice carries no security weight. It is frozen
because it is load-bearing through *identity*: two ports that read a different
window print different numbers for the same phrase, and the number is what two
people read to each other during a support handoff or a self-host setup. The
number is not a secret and it is never part of a recovery kit.

## Version note

This family lives under `v0.2/`, not `v0.1/`. The *scheme* is the v0.1 one, but
the input is a v0.2 value: a v0.1 client has no person identity, no person id
and no phrase-to-person hash. `person-keypair` and `person-identity` sit here
for the same reason.

## File layout

```
vectors/person-account-number/
├── schema.md              ← this file
└── v0.2/
    └── cases.json         ← 5 cases, one `case_kind` (`derive`)
```

## JSON shape

The machine-readable contract lives at
[`schema/v0.2.json`](../../schema/v0.2.json) plus this category's sub-schema
[`schema/v0.2/person-account-number.json`](../../schema/v0.2/person-account-number.json).

- `input_person_id_hex` is the **normative** input: this family owns the hop
  from person id to number.
- `input_phrase_utf8_hex` is carried on the cases that start from a phrase, so a
  port can also pin the first hop (`person_id_hash`). Present or absent, never a
  second source of truth: the runner cross-checks whichever is there.
- `expected_canonical_ascii` / `expected_display_ascii` are ASCII literals, not
  hex, because they are compared as text.
- `expected_account_number_hash_hex` is `SHA-256(canonical_ascii)[:16]`.

## The two hand-computable cases

`derive-all-zero-id` and `derive-top-bit-id` are the only cases whose expected
value can be worked out on paper, and they are the only ones that can catch a
swapped window (a low-95-bit mask), a little-endian read or a moved bit order.
The runner asserts BOTH are present, so the family cannot quietly degrade into a
transcript of whatever our implementation printed.

`derive-below-window-id` is the third guard: an id with nothing set above bit 32
must produce the all-zero canonical body, which a low-95 mask would not.

## Regeneration

```bash
cd ../myne && cargo test -p protocol --lib emit_person_account_number_vectors_for_paste -- --ignored
# then paste /tmp/myne_person_account_number_vectors.json over this directory's cases.json
```

The emitter asserts the orientation cases at mint, so a wrong window fails the
GENERATOR rather than being frozen as truth.
