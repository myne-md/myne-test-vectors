# group-roster (v0.2)

The owner-signed group roster, and the two signature classes it can carry
(`docs/spec/protocol.md` §9.3.3; main repo `decisions/protocol/0731` stage 2 and
`0733`).

## Why this category exists

The roster was person-signed and nothing else could vouch for it, so inviting and
removing a member needed the owner's recovery phrase on every device. Stage 2 moves that
signature to the owner's **device** key. The payload does not move at all: it still names
PEOPLE and still carries the owner's person fingerprint, and the frame is still
`payload_len (u32 BE) || payload || ed25519_sig (64) || ml_dsa_65_sig (3309)`. What
moves is the class prefix, from `"myne.group-roster v1|"` to `"myne.group-roster v2|"`,
which is what a reader dispatches on.

## What a reader MUST do

Accept both classes. Resolve the device from an attestation it trusts: the `owner_device`
field carried inside a person-signed group package, its own device key, or a verified
`vault.device-bindings` row that names the owner. A reader with none of those refuses a
`v2` roster, because a roster carries no key of its own.

## Case kinds

| kind | what it pins |
|---|---|
| `signed_bytes` | The exact message each class signs: `class prefix || canonical payload`. The prefix is inside the signed bytes, which is why a person signature can never satisfy a device verifier. |
| `verify` | Four verdicts over WHOLE frames, so a port exercises its own decoder: `v1` with the person key, `v2` with the vouched device, `v2` with no attestation (refused), `v2` with a different device (refused). |
| `package_payload` | One recipient's canonical group-key package payload carrying the `owner_device` attestation, and the same payload from a build with no device key, where the field is ABSENT rather than null. That absence is what makes the field additive rather than a format break. |

## The attestation, in one paragraph

A member's first roster verification happens before it holds a group key, so it cannot
read the binding store. The owner's device halves therefore travel inside the
person-signed package as `owner_device`: the device id and the owner's person-signed
binding row. The row travels WHOLE rather than as two loose halves so the recipient
verifies the person's signature over the row itself, against the owner public half it
pinned out of band, instead of trusting the package's word that those halves belong to
the owner.

## Regeneration

```bash
cargo test -p protocol --lib emit_group_roster_vectors_for_paste -- --ignored --nocapture
```

Its output replaces `vectors/group-roster/v0.2/cases.json` in one act, so read the diff
before committing it. The fixture is deterministic: the generator drives the signature
RNG from fixed seeds, and the owner's public halves are pinned at the top level under
`informational_owner` so the runner can verify a frame it did not author.
