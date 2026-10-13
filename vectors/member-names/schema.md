# `member-names` vector schema

Pins the byte mapping of `protocol::encode_member_names_blob_raw` +
`protocol::decode_member_names_blob` (Rust crate `protocol`, in the
[main `myne` repo](https://github.com/myne-md/myne)). The codec emits and consumes
`vault.member-names` — the vault singleton holding the group's member **display
names**, one row per `person_id_hash` (`MYO5` magic; myne-md/myne#851).

**Reference implementation:** `protocol/src/member_names_blob.rs`.
**Spec:** `docs/spec/protocol.md` § 9.29 (the `vault.member-names` envelope, the
canonical map, the caps, the write-side guards) + § 3.4 (the two GROUP-tier subkey
roles) + § 5.2 (the AAD role and the `member-names-blob` builder).
**Decisions:** `decisions/protocol/0666` (the family: a group-tier-sealed `MYO5`
map) and `decisions/protocol/0667` (the `myne.member-name v1|` signature class).

## Why the family is `NTV6`-only

`vault.member-names` exists only inside a B2B vault, and three consequences of that
are load-bearing for a port:

1. **There is no `v0.1/` side and no v0.1 form of the magic.** A pre-B2B reader must
   refuse `MYO5` on sight. `MYO5` names bytes on disk today; reading the *next*
   letter in the pool as "unallocated" is the mental model behind myne-md/myne#638,
   where a reader taught only one generation's magic returned `None` for every body
   and a whole subsystem stopped working in both directions while every gate stayed
   green.
2. **Neither role has a `SubkeyRole` counterpart.** `GroupRole::MemberNamesKey` and
   `GroupRole::MemberNamesSyncId` are GROUP-tier roles, keyed from the vault's
   `group_key`, not from `root_key`. `SubkeyRole::shared_group_role()` returns `None`
   for 15 of its variants and must keep returning `None` for a role that has no
   root-tier twin — a member-names key reachable from `root_key` would be readable by
   an owner-only reader, which is exactly what the group tier exists to prevent. That
   is why the two derivations are pinned **inside this family** (the `group_subkey`
   case kind) rather than in the `hkdf-sha256` category: that category's runner and
   its `input_role` enum are both keyed on
   `derive_subkey_v0_2(root_key, SubkeyRole)`, and neither role can enter it.
   `protocol/tests/vectors.rs` states this as an assertion rather than a comment: the
   `member_names_vectors` runner iterates `SubkeyRole::ALL`, maps each variant through
   `shared_group_role()`, and fails by name if either member-names role appears in
   that projection.
3. **The blob is group-sealed, not owner-sealed**, so a member reads and writes it.
   Everything below follows from that: the row signature is what keeps a member from
   editing a co-member's name.

## The two role strings

A porter must not conflate the two, and must not invent a third:

- **KEK (GROUP/HKDF) role** `"member-names-key"` — `member_names_key =
  derive_group_subkey_v0_2(group_key, GroupRole::MemberNamesKey)`.
- **AAD role** `"member-names-blob"` (17 ASCII bytes, length-prefixed) — the role
  inside `aad_member_names_blob(account_number_hash, group_key_epoch)`.

`GroupRole::MemberNamesSyncId` (`"member-names-sync-id"`) derives the reserved
16-byte sync blob-id of `vault.member-names` for Phase 2 sync. It is an **identifier**
derivation, not a KEK; the KEK stays `member-names-key`. Both are pinned here as
`group_subkey` cases with a non-vacuity check that the two derived keys differ.

## The AAD binds the group epoch

`aad_member_names_blob` is **54 bytes**, in this order:

| offset | length | segment |
|---|---|---|
| 0 | 10 | `"myne.v0.2|"` (the v0.2 AAD prefix) |
| 10 | 1 | `protocol_version` `0x02` |
| 11 | 2 | role length, big-endian (`0x0011` = 17) |
| 13 | 17 | `"member-names-blob"` |
| 30 | 16 | `account_number_hash` |
| 46 | 8 | `group_key_epoch`, big-endian |

The **`group_key_epoch` is the segment that is specific to this family** and it is
required, not optional: a post-rotation blob must fail rather than be read under a
key that no longer applies. Without it, a member removed by a group rotation would
keep reading the name map under the key they still hold. The decomposition is
pinned segment-by-segment in the `aad-construction/v0.2` category
(`aad_member_names_blob`), and the full bytes here.

**Epoch zero is refused on both sides before anything else runs.** `0` is a value no
writer produces, so it is a distinguishable error (`epoch_zero`) rather than falling
into the non-distinguishing AEAD surface — the check is a header/argument-level
guard, not a decryption outcome. It is pinned three times: a `decrypt_fail` case, an
`encode_fail` case, and as the `input_group_key_epoch` minimum in this category's
sub-schema.

## Outer layout

Fixed-offset §9.2 envelope, `MEMBER_NAMES_BLOB_HEADER_LEN = 34` bytes plus the
ciphertext:

| offset | length | field |
|---|---|---|
| 0 | 4 | magic `MYO5` |
| 4 | 1 | `protocol_version` `0x01` |
| 5 | 1 | `format_version` `0x01` (versions the inner map schema) |
| 6 | 24 | `nonce` |
| 30 | 4 | `ciphertext_len`, big-endian u32 (`plaintext.len() + 32`) |
| 34 | … | ciphertext + 32-byte CTX commitment tag |

No new `SubkeyRole`; the §9.2 envelope with its magic, AAD role and version bytes
swapped. **The `protocol_version` byte is `0x01` even though the AAD prefix is
`"myne.v0.2|"`.** That is not an inconsistency to "correct": the header byte gates
the *envelope generation* of this family's own layout, and the family was born in
v0.2, so v0.2's own generation number is the only one it has ever had. The v0.2 AAD
prefix is what separates it from a v0.1 family. The sub-schema pins both bytes as
`const` for exactly this reason.

The whole-map plaintext cap is **4 MiB** (`MAX_MEMBER_NAMES_PLAINTEXT_LEN`), sized by
arithmetic rather than by feel: one row is a name (up to 64 bytes) plus a hybrid
signature hex-encoded at 6746 characters, so a row is ~6.8 KiB and the cap holds
around 600 members.

## The inner map is canonical JSON, not opaque

Unlike `vault.themes` / `vault.search` / `vault.workspace`, this family owns its inner
shape, because the map is a **merge** surface: two devices must agree on the bytes of
the same logical map or one of them will keep re-uploading.

- `person_id_hash` → row, lowercase hex keys, **byte-ascending**.
- Row fields in declaration order: `name`, `version`, `tombstone`, `signature_hex`.
  No whitespace, no trailing newline.
- The decoder **re-encodes what it parsed and requires byte equality**
  (`MapNotCanonical` otherwise), so a non-canonical-but-well-formed input is refused
  rather than silently accepted and later re-emitted in a different form. Vector
  string: `not_canonical`.
- A malformed map — bad JSON, a key that is not a 32-hex `person_id_hash`, a name over
  the cap, an empty name on a live row, a tombstone carrying a name, a signature that
  is not exactly 6746 lowercase hex characters — is a **plaintext-level parse failure**
  after the AEAD has already opened. Vector string: `plaintext_parse_failed` (the name
  the sibling `vault-prefs` / `attachments-catalog` families already use for this, so
  an independent implementation maps one failure to one name across families).

**A tombstone row is KEPT, not deleted.** Dropping it would let a stale copy
re-introduce the name on the next merge. `live_member_names` is the view a member list
renders.

### The merge rule (pinned by the codec, not by this category's cases)

Higher `version` wins; at **equal** version a tombstone beats a live entry, so a
concurrent rename cannot resurrect a removal. Ties this cannot separate (equal
version, equal tombstone flag) keep the incumbent, which makes the merge stable and
order-independent. `merge_member_names` is commutative and idempotent.

This category does **not** carry merge cases, and that is a scope decision rather
than an omission: the roster/turn-list join vectors for `agent-memory-merge` and
`chats-merge` live in their own sibling categories, and the name map's merge belongs
in the same shape (`vectors/member-names-merge/`) if it earns one. What is pinned here
is the byte-level codec contract the merge is defined over.

## The signature covers a fixed-width message, not the JSON

`member_name_signed_bytes` is the **message class** the row's hybrid signature covers:

```
"myne.member-name v1|"  (20 bytes, the registered class prefix)
|| person_id             (16 bytes — the person the row NAMES)
|| version               (8 bytes, big-endian)
|| tombstone             (1 byte: 0x00 live / 0x01 removed)
|| name                  (UTF-8, to the end of the message)
```

Fixed widths rather than canonical JSON, so the signed message has exactly one
encoding and there is no ambiguity about where a field ends. The signature bytes are
Ed25519 (64) || ML-DSA-65 (3309) = **3373 bytes / 6746 hex characters**.

**The signature BYTES are deliberately not pinned anywhere in this repository.** The
ML-DSA-65 half is *randomized* — `PersonKeyPair::sign` consumes the caller's RNG — so
a pinned signature would only be reproducible by a port that also reproduced our exact
RNG stream, which is not the contract. What a port must reproduce is the MESSAGE,
which is why `member_name_signed_bytes` is `pub` and why the three `signed_bytes`
cases pin it directly, including `informational_name_len_bytes` so a port that
length-prefixes the name fails immediately.

What the `verify` cases pin instead is the **verification outcome**, against real
signatures:

- a row signed by the person it names verifies;
- a row whose `name`, `version` or `tombstone` was altered after signing does **not**;
- a valid signature presented against a **different person's public half** does not,
  and neither does the same valid signature checked against a **different
  `person_id`** with the same key — so the id inside the signed message is what binds
  a row to the member it names, not the key alone. Without that binding a member could
  sign once and have the row appear under any co-member's name;
- an **owner-signed tombstone** verifies — that is how a removal the owner performed
  clears a row;
- that same owner-signed tombstone checked against the public half of the member the
  row **names** does **not** verify, which is why `partition_verified` takes the
  owner's public half as a SEPARATE argument and reaches for it only on a tombstone;
- an owner signature over a **live** row does **not**: the removal check refuses
  anything that is not a tombstone before it looks at the signature, so an owner can
  never use the removal path as a name-writing path.

`partition_verified` splits a map into verified rows and the ids that failed (each
tombstone additionally accepting the owner's public half), and **returns** the failed
ids rather than dropping them, so a caller can surface a tampered row as unverified
instead of silently treating it as absent.

Every `verify` case carries the signer's full public half
(`input_signer_kem_ek_hex` / `input_signer_ed_pk_hex` / `input_signer_mldsa_vk_hex`),
and the generator asserts the published triple rebuilds the very key that produced the
pinned outcome — so a case cannot pin an outcome that no longer describes the key
beside it.

## File layout

```
vectors/member-names/
└── v0.2/
    └── cases.json         ← array of 35 cases, 7-way case_kind
```

No `v0.1/`: the family does not exist below v0.2.

## JSON shape

The machine-readable contract lives at [`schema/v0.2.json`](../../schema/v0.2.json)
(top-level shape) and [`schema/v0.2/member-names.json`](../../schema/v0.2/member-names.json)
(per-case shape, a 7-way `case_kind` discriminator). Run the repository's documented
`ajv-cli validate` command to verify any `cases.json` against the canonical schema.
The sub-schema is authoritative — this document covers only the domain-specific
semantics.

| `case_kind` | cases | what it pins |
|---|---|---|
| `encrypt` | 5 | the encoder's byte output + the header facts the format fixes (`expected_magic_ascii`, both version bytes, `informational_header_len`, `informational_nonce_offset`) |
| `decrypt_fail` | 5 | the non-distinguishing AEAD surface (§6.4), one input wrong per case |
| `parse_fail` | 8 | the pre-AEAD header refusals, plus the two inner-map refusals |
| `encode_fail` | 3 | the write-side guards (the cap, the tombstone-with-a-name rule, epoch zero) |
| `signed_bytes` | 3 | the exact message a row's signature covers |
| `verify` | 9 | the acceptance rules, as outcomes |
| `group_subkey` | 2 | the two GROUP-tier derivations the family is keyed by |

### `encrypt`

Inputs: `input_member_names_key_hex` (32 B), `input_account_number_hash_hex` (16 B),
`input_group_key_epoch` (integer), `input_nonce_hex` (24 B), `input_map` (the map as
the canonical JSON object it is sealed as). Output: `expected_blob_hex`.

The five cases are: a two-row map with one live row and one tombstone; the **empty
map** — 2 bytes of plaintext (`{}`), so a 68-byte blob, and the form a vault that has
never had a display name stores, deliberately distinguishable from an ABSENT blob
(which `decode_member_names_or_empty` also collapses to the empty map, but which is not
this byte string); an ASCII name **at** the 64-byte cap and a multibyte name at the same
cap (32 two-byte characters = 64 bytes, which pins the cap as counting **UTF-8 bytes**,
not characters or code points — a port that counts code points accepts a 100-byte
name); and a row carrying a **real** signature, so the codec is pinned against bytes
`verify_member_name` accepts rather than only against placeholders.

### `decrypt_fail`

Output: `expected_error`. Four of the five are `cannot_decrypt` and **must be
indistinguishable from each other**: a tampered CTX tag, a cross-vault
`account_number_hash`, a **stale `group_key_epoch`**, and a wrong key. Fixtures carry
the REAL key, so a port that skips a check surfaces as a spurious `Ok` rather than a
coincidental AEAD failure. The fifth, `epoch_zero`, is deliberately NOT collapsed into
that set — it is an argument-level refusal, not a decryption outcome.

### `parse_fail`

Output: `expected_error`, one of `header_too_short`, `magic_mismatch`,
`protocol_version_mismatch`, `format_unsupported`, `ciphertext_len_overflow`,
`ciphertext_truncated`, and the two post-AEAD map refusals `plaintext_parse_failed` /
`not_canonical`. These are **distinguishable** and run before decryption (except the
last two), which is what lets a client fall back to the empty map cleanly.
`format_unsupported` is the forward hook: a future `format_version` must be rejected
by name so an older client refuses rather than half-parsing.

### `encode_fail`

Output: `expected_error`. Three write-side guards that have no decode counterpart:
a name one byte over the cap, a tombstone carrying a non-empty name, and
`group_key_epoch = 0`. Each is a *refusal to produce bytes*, which is why it is its
own kind — a decode-a-blob case cannot express it.

### `signed_bytes`

Inputs: `input_person_id_hex`, `input_version`, `input_tombstone`, `input_name_utf8`.
Outputs: `expected_message_hex`, `expected_prefix_ascii`, and informational
`informational_name_len_bytes`. The three cases are a live row, a tombstone with an
empty name, and a name at the byte cap.

### `verify`

Inputs: `verifier` (`person` or `owner`, i.e. whose public half to check against),
`input_person_id_hex`, the signer's three public-half parts, `input_entry` (the row).
Outputs: `expected_verified` (boolean) and informational
`informational_signed_bytes_hex`, which the runner cross-checks against
`member_name_signed_bytes` so the message and the outcome cannot drift apart.

### `group_subkey`

Inputs: `input_group_key_hex` (32 B), `input_role`
(`member-names-key` / `member-names-sync-id`). Output: `expected_subkey_hex`.

## Invariants the vectors pin

- **`MYO5` + both version bytes, checked before any decryption.** `MYO5` is its own
  blob family; a v0.1-only reader must refuse it.
- **Header byte layout and nonce offset.** `[6..30]` is the supplied nonce,
  `[30..34]` the big-endian `ciphertext_len`, `[34..]` ciphertext + 32-byte CTX tag.
- **The AAD binds the epoch.** Two blobs identical but for `group_key_epoch` MUST NOT
  open under each other's key — that is the whole point of the last AAD segment, and
  the `decrypt-fail-stale-epoch` case is what holds it.
- **The empty map round-trips** to 66 bytes and decodes back to empty.
- **The name cap counts UTF-8 bytes** and is inclusive at 64.
- **Canonical bytes are the only accepted bytes.** `not_canonical` is a refusal, not a
  normalization.
- **The message class is fixed-width**, so the signed bytes have exactly one encoding.
- **Signatures are not pinned; outcomes are.**
- **The two member-names roles have no root-tier counterpart**, checked against
  `SubkeyRole::ALL` rather than asserted in prose.

## How to consume

A port dispatches on `case_kind`:

- **`encrypt`**: derive `member_names_key` from `input_group_key_hex`, invoke the
  port's `encode_member_names_blob_raw` equivalent with the supplied nonce, epoch,
  account hash and map, hex-encode, compare byte-equal to `expected_blob_hex`.
- **`decrypt_fail`**: invoke the decoder, assert the *named* failure. For the four
  `cannot_decrypt` cases the port MUST NOT distinguish which arm fired.
- **`parse_fail`** / **`encode_fail`**: assert the specific error named by
  `expected_error`.
- **`signed_bytes`**: rebuild the message with the port's own assembly, compare
  byte-equal.
- **`verify`**: build the signer's public half from the three published parts, verify
  the row's signature over the message, compare to `expected_verified`. For
  `verifier: "owner"` use `verify_member_removal`; for `"person"` use
  `verify_member_name`.
- **`group_subkey`**: derive the GROUP-tier subkey for `input_role` from
  `input_group_key_hex`, compare byte-equal.

Any mismatch is a stop-the-world bug (per repo `README.md` "Cross-implementation
discipline").

## Regeneration

```bash
cargo test -p protocol --lib emit_member_names_vectors_for_paste -- --ignored --nocapture
```

The generator lives in `protocol/src/member_names_blob.rs` and is `#[ignore]`-gated,
so it never runs in a default `cargo test`. It self-asserts every expectation at mint
(header facts, canonical round-trip, the epoch refusal on both sides, the
public-half round-trip) and prints the whole file.

**The committed file IS reproducible, verified rather than assumed.** Two independent
runs of the command above produce identical output, because the generator drives
`PersonKeyPair::sign`'s randomized ML-DSA-65 half from a fixed-seed `SeededRng`: the
signature bytes are a function of the seed and the message and nothing else. The command
prints a document, and the committed file is that document parsed and re-emitted at a
two-space indent (this family's files are fully expanded; the sibling `device-binding`
family pastes its emitter's output verbatim, so match whichever one you are regenerating).
This paragraph said the opposite for a round ("a snapshot, not a reproducible artefact"),
which was a plausible-sounding claim about a randomized signature scheme that nobody had
run twice; the emitter's fixed seed is what makes it false.

Two things have to stay true together, and both are one command to check. Re-running the
emitter must reproduce the file, and the file must not carry text the emitter has stopped
emitting. The second half was false here: this file held eight `description` strings with
an em-dash where the emitter now writes a comma, and regenerating it cleared them. An
emitter that reached for the OS CSPRNG would break the first silently, so re-run the
reproducibility diff after touching the generator.

Diff the emitted ids and expected bytes against the committed file case-by-case rather
than comparing counts: editing an emitter can drop one case and mint another in its
place while `cases.len()` stays green.

Runner: `member_names_vectors` (`protocol/tests/vectors.rs`). The category also has a
row in that file's `mod embedded` table, which is what the wasm/browser lane reads.

## What this does NOT cover

- **The merge.** See "The merge rule" above — merge semantics belong in their own
  sibling category, alongside `agent-memory-merge` / `chats-merge`.
- **Group rotation.** Neither `group_rotation_reclaim_sync_ids` nor
  `SHARED_SINGLETONS` in `myne-core` names `vault.member-names` yet, so the reclaim and
  the re-seal under the new epoch are **unwired**. Until they land together, a member
  removal leaves the old-epoch name blob readable by the just-removed member while the
  new epoch has no readable name store. That is a code obligation in the main repo, not
  a vector gap — the vectors pin the epoch binding that makes the fix meaningful.
- **The `getrandom` convenience wrapper** (`encode_member_names_blob`) — every family
  vectors only its deterministic `_raw` sibling.
- **Client-layer fallback discipline** (absent file / truncated / AEAD failure /
  unsupported format collapsing to "no names yet", via `decode_member_names_or_empty`)
  — wrapper behaviour, verified by `myne-core` integration tests.
- **The roster.** Who the members are, and where a `person_id_hash` resolves to a
  public half, is the membership feature's surface; this family stores names.
