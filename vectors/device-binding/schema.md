# `device-binding` vector schema

Pins the byte mapping of `protocol::encode_device_binding_blob_raw` +
`protocol::decode_device_binding_blob` (Rust crate `protocol`, in the
[main `myne` repo](https://github.com/myne-md/myne)). The codec emits and consumes
`vault.device-bindings`: the vault singleton holding the group's device-to-person
**bindings**, one row per `device_id` (`MYU5` magic; the `member-devices` feature).

**Reference implementation:** `protocol/src/device_binding_blob.rs`.
**Spec:** `docs/spec/protocol.md` § 9.30 (the `vault.device-bindings` envelope, the
canonical map, the merge rule, the write-side guards) + § 3.4 (the GROUP-tier subkey
role) + § 5.2 (the AAD role and the `device-binding-blob` builder).
**Decisions:** `decisions/protocol/0669` (the family: a group-tier-sealed `MYU5`
map) and `decisions/protocol/0670` (the `myne.device-binding v1|` signature class).

## Why the family is `NTV6`-only

`vault.device-bindings` exists only inside a B2B vault, and three consequences of
that are load-bearing for a port:

1. **There is no `v0.1/` side and no v0.1 form of the magic.** A pre-B2B reader must
   refuse `MYU5` on sight. `MYU5` names bytes on disk today; reading the *next*
   letter in the pool as "unallocated" is the mental model behind myne-md/myne#638,
   where a reader taught only one generation's magic returned `None` for every body
   and a whole subsystem stopped working in both directions while every gate stayed
   green.
2. **Neither GROUP-tier role has a `SubkeyRole` counterpart.** `GroupRole::DeviceBindingKey`
   is a GROUP-tier role, keyed from the vault's `group_key`, not from `root_key`, and so
   is `GroupRole::DeviceBindingSyncId`, the deterministic blob-id the singleton is
   published at. A pre-B2B vault has no `group_key`, so there is no vault in which a
   member-writable binding map could exist at all. `SubkeyRole::shared_group_role()`
   answers `Some` for 15 of its 29 variants and `None` for the other 14, and must keep
   answering `None` for both of these: a device-binding key reachable from `root_key`
   would be readable by an owner-only reader, which is exactly what the group tier
   exists to prevent. That is why both
   derivations are pinned **inside this family** (the `group_subkey` case kind) rather
   than in the `hkdf-sha256` category: that category's runner and its `input_role`
   enum are both keyed on `derive_subkey_v0_2(root_key, SubkeyRole)`, and neither role
   can enter it. `protocol/tests/vectors.rs` states this as an assertion rather
   than a comment: the `device_binding_vectors` runner iterates `SubkeyRole::ALL`,
   maps each variant through `shared_group_role()`, and fails by name if either
   appears in that projection.
3. **The blob is group-sealed, not owner-sealed**, so a member reads and writes it.
   Everything below follows from that. The seal buys confidentiality and nothing
   else: anyone who can edit the blob can write any row, which is why each row
   carries its own signature and why the signature is checked against the person the
   row CLAIMS.

## The role strings

A porter must not conflate them, and must not invent a fourth:

- **KEK (GROUP/HKDF) role** `"device-binding-key"`: `device_binding_key =
  derive_group_subkey_v0_2(group_key, GroupRole::DeviceBindingKey)`.
- **Sync blob-id (GROUP/HKDF) role** `"device-binding-sync-id"`:
  `device_binding_sync_id = derive_group_subkey_v0_2(group_key,
  GroupRole::DeviceBindingSyncId)[0..16]`, the address the singleton is published at.
  It derives an **identifier, not key material**, and that distinction is load-bearing
  rather than cosmetic: it is the id that ROTATES with the group key when a member
  is removed, which is what makes the retired epoch's record reclaimable while the
  new epoch's blob stays unreadable to the member just removed.
- **AAD role** `"device-binding-blob"` (19 ASCII bytes, length-prefixed): the role
  inside `aad_device_binding_blob(account_number_hash, group_key_epoch)`.

The role strings are distinct from the sibling `member-names` family's
(`member-names-key` / `member-names-sync-id` / `member-names-blob`) even though both
families are member-writable and both are keyed from the same group key: two families,
two KEKs, so a blob of one can never be opened as the other. The `member-names`
category makes the same three-way split, so a port that reads one of the two families
has the shape for the other.

## The AAD binds the group epoch

`aad_device_binding_blob` is **56 bytes**, in this order:

| offset | length | segment |
|---|---|---|
| 0 | 10 | `"myne.v0.2|"` (the v0.2 AAD prefix) |
| 10 | 1 | `protocol_version` `0x02` |
| 11 | 2 | role length, big-endian (`0x0013` = 19) |
| 13 | 19 | `"device-binding-blob"` |
| 32 | 16 | `account_number_hash` |
| 48 | 8 | `group_key_epoch`, big-endian |

The **`group_key_epoch` is the segment that is specific to this family** and it is
required, not optional: a post-rotation blob must fail rather than be read under a
key that no longer applies. Without it, a member removed by a group rotation would
keep reading the binding map under the key they still hold. The decomposition is
pinned segment-by-segment in the `aad-construction/v0.2` category
(`aad_device_binding_blob`), and the full bytes here.

**Epoch zero is refused on both sides before anything else runs.** `0` is a value no
writer produces, so it is a distinguishable error (`epoch_zero`) rather than falling
into the non-distinguishing AEAD surface: the check is an argument-level guard, not
a decryption outcome. It is pinned three times: a `decrypt_fail` case, an
`encode_fail` case, and as the `input_group_key_epoch` minimum in this category's
sub-schema.

## Outer layout

Fixed-offset §9.2 envelope, `DEVICE_BINDING_BLOB_HEADER_LEN = 34` bytes plus the
ciphertext:

| offset | length | field |
|---|---|---|
| 0 | 4 | magic `MYU5` |
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

The whole-map plaintext cap is **4 MiB** (`MAX_DEVICE_BINDINGS_PLAINTEXT_LEN`), sized
by the same arithmetic as the sibling: one row is a 32-byte fingerprint plus a hybrid
signature hex-encoded at 6746 characters, so a row is ~6.8 KiB and the cap holds
around 600 devices.

## The inner map is canonical JSON, not opaque

Unlike `vault.themes` / `vault.search` / `vault.workspace`, this family owns its inner
shape, because the map is a **merge** surface: two devices must agree on the bytes of
the same logical map or one of them will keep re-uploading.

- `device_id` → row, lowercase hex keys, **byte-ascending**.
- Row fields in declaration order: `person_fingerprint`, `version`, `tombstone`,
  `signature_hex`. No whitespace, no trailing newline.
- The decoder **re-encodes what it parsed and requires byte equality**
  (`MapNotCanonical` otherwise), so a non-canonical-but-well-formed input is refused
  rather than silently accepted and later re-emitted in a different form. Vector
  string: `not_canonical`.
- A malformed map: bad JSON, a key that is not a 32-hex `device_id`, a
  `person_fingerprint` that is not 64 lowercase hex characters, a signature that is
  not exactly 6746 lowercase hex characters, is a **plaintext-level parse failure**
  after the AEAD has already opened. Vector string: `plaintext_parse_failed` (the name
  the sibling `vault-prefs` / `attachments-catalog` / `member-names` families already
  use for this, so an independent implementation maps one failure to one name across
  families). The write side runs the same check and refuses, so an invalid map is
  never written and then refused on the next read.

**The map key is the DEVICE, not the person**, and that is the family's one
structural departure from its `member-names` sibling (whose key is the person). A
device that moves between members keeps its key and gets a new row, which is what
makes a re-bind an ordinary write. It also means the claimed person lives *inside*
the row, which is what forces the extra check in `verify_device_binding` below.

**A tombstone row is KEPT, not deleted.** Dropping it would let a stale copy
re-introduce the binding on the next merge. Its `person_fingerprint` is kept too, so
a removal still names the binding it records, unlike the `member-names` tombstone,
which carries no name. `live_device_bindings` is the view a device list renders.

## The merge rule (pinned by the codec AND by this category)

`merge_device_bindings(a, b)` is per-device and resolves each row with a total order:

1. Higher `version` wins.
2. At equal `version`, a **tombstone beats a live entry**, so a concurrent re-claim
   cannot resurrect a removed device.
3. At equal version and equal tombstone flag, the greater `person_fingerprint` bytes
   win, then the greater `signature_hex` bytes. Rust's `String` ordering is byte-wise
   and a port must keep it byte-wise rather than reach for a locale collation, or the
   two sides would pick different winners for the same pair.

With those arms the relation is a total order on distinct rows, which is what makes
the merge commutative, associative and idempotent rather than merely stable.

**One pair has no winner, and the merge refuses it rather than inventing one.** Two
copies carrying the SAME `device_id` bound to two DIFFERENT people at the SAME version
are an unresolvable tie: every remaining field is either equal (the version) or the
thing under dispute (the person), so picking a side would make `merge(a, b)` and
`merge(b, a)` disagree, each peer would keep its own row and re-push it every cycle,
and the device would sit under two members forever with no convergence.
`DeviceBindingBlobError::ConflictingBinding` names it instead, and the caller decides
(a fresh claim at a strictly higher version is the repair). The refusal is symmetric,
so it does not itself break order independence: a pair either merges to the same map
both ways or fails both ways.

**This is a case kind that lives inside a family's own codec category, and no earlier
family has needed one.** Merges are pinned in this repository as whole CATEGORIES of
their own, one per merge surface (`agent-memory-merge` and `chats-merge`, both on the
v0.1 side), because those merges join rosters the family itself does not own. Here the
merge is a property of the codec's own inner map, so it belongs in the category that
pins that map. The three `merge` cases are the shape: a re-bind that resolves by
version, the same-version-two-people pair that is refused, and a tombstone that beats a
live entry at an equal version. Because one of the three pins a REFUSAL rather than a
result, this is also the only category in the repository whose sub-schema needs a nested
`oneOf` inside one `case_kind` branch (see "JSON shape").

## The signature covers a fixed-width message, not the JSON

`device_binding_signed_bytes` is the **message class** the row's hybrid signature
covers:

```
"myne.device-binding v1|"  (23 bytes, the registered class prefix)
|| device_id               (16 bytes, the MAP KEY)
|| person_fingerprint      (32 bytes, the person the row CLAIMS)
|| version                 (8 bytes, big-endian)
|| tombstone               (1 byte: 0x00 live / 0x01 removed)
```

Fixed widths rather than canonical JSON, so the signed message has exactly one
encoding and there is no ambiguity about where a field ends. **80 bytes**, constant:
the fingerprint is present on a tombstone too, so unlike `member-names` the length
does not vary with the tombstone flag. The signature bytes are Ed25519 (64) ||
ML-DSA-65 (3309) = **3373 bytes / 6746 hex characters**.

**Signature bytes are inputs a port CONSUMES, never an output it must reproduce.**
`PersonKeyPair::sign`'s ML-DSA-65 half is randomized and takes the caller's RNG, so no
port can be asked to mint our exact bytes; the generator happens to use a fixed seed
which is why the file is reproducible for *us* (see "Regeneration"), but that is an
emitter-side property and not part of the contract. What a port must reproduce is the
MESSAGE, which is why `device_binding_signed_bytes` is `pub` and why the three
`signed_bytes` cases pin it directly.

What the `verify` cases pin instead is the **verification outcome**, against real
signatures:

- a row signed by the person it claims verifies;
- a row whose `version`, `person_fingerprint` or `tombstone` was altered after signing
  does **not**: each of the three is inside the message, and each is load-bearing:
  the version decides the merge, the fingerprint is what the whole family exists to
  carry, and the tombstone flag is a device removal;
- a valid signature presented against a **different person's public half** does not,
  and neither does the same valid signature checked against a **different `device_id`**
  with the same key, so both fields inside the signed message bind, not the key alone;
- a row whose claimed `person_fingerprint` does not equal the **signer's own**
  fingerprint does not verify. This check is `verify_device_binding`'s addition over
  `verify_member_name` and it is load-bearing here rather than decoration: the
  sibling's map key IS the person, so a row is only ever looked up under its own
  author, whereas here the key is the DEVICE, so without this check a member could sign
  one message carrying a co-member's fingerprint and file it under themselves;
- an **owner-signed tombstone** verifies: that is how a removal the owner performed
  clears a device;
- that same owner-signed tombstone checked against the public half of the member the
  row **claims** does **not** verify, which is why `partition_verified_bindings` takes
  the owner's public half as a SEPARATE argument and reaches for it only on a
  tombstone;
- an owner signature over a **live** row does **not**: the removal check refuses
  anything that is not a tombstone before it looks at the signature, so an owner can
  never use the removal path as a binding-writing path.

`partition_verified_bindings` splits a map into verified rows and the ids that failed
(each tombstone additionally accepting the owner's public half), and **returns** the
failed ids rather than dropping them, so a caller can surface a tampered row as
unverified instead of silently treating it as absent.

Every `verify` case carries the signer's full public half
(`input_signer_kem_ek_hex` / `input_signer_ed_pk_hex` / `input_signer_mldsa_vk_hex`),
and the generator asserts the published triple rebuilds the very key that produced the
pinned outcome, so a case cannot pin an outcome that no longer describes the key
beside it. The `kem_ek` half is published even though verification reads only the
Ed25519 and ML-DSA halves, so a port builds the same struct the roster carries rather
than inventing a placeholder for a field the real code path fills.

## File layout

```
vectors/device-binding/
└── v0.2/
    └── cases.json         ← array of 35 cases, 8-way case_kind
```

No `v0.1/`: the family does not exist below v0.2.

## JSON shape

The machine-readable contract lives at [`schema/v0.2.json`](../../schema/v0.2.json)
(top-level shape) and [`schema/v0.2/device-binding.json`](../../schema/v0.2/device-binding.json)
(per-case shape, an 8-way `case_kind` discriminator). Run the repository's documented
`ajv-cli validate` command to verify any `cases.json` against the canonical schema.
The sub-schema is authoritative: this document covers only the domain-specific
semantics.

| `case_kind` | cases | what it pins |
|---|---|---|
| `encrypt` | 3 | the encoder's byte output + the header facts the format fixes (`expected_magic_ascii`, both version bytes, `informational_header_len`, `informational_nonce_offset`) |
| `decrypt_fail` | 5 | the non-distinguishing AEAD surface (§6.4), one input wrong per case, plus the distinguishable epoch-zero guard |
| `parse_fail` | 8 | the pre-AEAD header refusals, plus the two inner-map refusals |
| `encode_fail` | 2 | the write-side guards (a malformed fingerprint, epoch zero) |
| `merge` | 3 | the per-device tie-break AND the one pair the merge must refuse |
| `signed_bytes` | 3 | the exact message a row's signature covers |
| `verify` | 9 | the acceptance rules, as outcomes, across both authorities |
| `group_subkey` | 2 | the two GROUP-tier derivations: the KEK the family is sealed under and the sync blob-id it is addressed by |

### `encrypt`

Inputs: `input_device_binding_key_hex` (32 B), `input_account_number_hash_hex` (16 B),
`input_group_key_epoch` (integer), `input_nonce_hex` (24 B), `input_map` (the map as
the canonical JSON object it is sealed as). Output: `expected_blob_hex`.

The three cases are: a two-row map with one live row and one tombstone; the **empty
map**: 2 bytes of plaintext (`{}`), so a 68-byte blob, and the form a vault that has
never bound a device stores, deliberately distinguishable from an ABSENT blob (which
`decode_device_bindings_or_empty` also collapses to the empty map, but which is not
this byte string); and a row carrying a **real** signature, so the codec is pinned
against bytes `verify_device_binding` accepts rather than only against placeholders.

### `decrypt_fail`

Output: `expected_error`. Four of the five are `cannot_decrypt` and **must be
indistinguishable from each other**: a tampered CTX tag, a cross-vault
`account_number_hash`, a **stale `group_key_epoch`**, and a wrong key. Fixtures carry
the REAL key, so a port that skips a check surfaces as a spurious `Ok` rather than a
coincidental AEAD failure. The fifth, `epoch_zero`, is deliberately NOT collapsed into
that set: it is an argument-level refusal, not a decryption outcome.

### `parse_fail`

Output: `expected_error`, one of `header_too_short`, `magic_mismatch`,
`protocol_version_mismatch`, `format_unsupported`, `ciphertext_len_overflow`,
`ciphertext_truncated`, and the two post-AEAD map refusals `plaintext_parse_failed` /
`not_canonical`. These are **distinguishable** and run before decryption (except the
last two), which is what lets a client fall back to the empty map cleanly.
`format_unsupported` is the forward hook: a future `format_version` must be rejected
by name so an older client refuses rather than half-parsing.

### `encode_fail`

Output: `expected_error`. Two write-side guards that have no decode counterpart: a
row whose `person_fingerprint` is not 64 lowercase hex characters, and
`group_key_epoch = 0`. Each is a *refusal to produce bytes*, which is why it is its
own kind: a decode-a-blob case cannot express it.

### `merge`

Inputs: `input_map_a`, `input_map_b`. Output: **EITHER** `expected_merged_map`
(the merge resolved it) **OR** `expected_error: "conflicting_binding"` (the merge
refused it), never both and never neither. That exclusivity is enforced by a nested
`oneOf` inside the `merge` branch of the sub-schema, which is the one shape a sibling
family has never needed: every other kind's arms are distinguished by their own
`case_kind`, so one branch per kind is enough, while here one kind carries two
mutually exclusive outcomes.

The three cases are a re-bind at a higher version (resolves to the later row), a
same-version pair naming two different people (refused), and a tombstone against a
live entry at an equal version (the tombstone wins). The runner merges every pair in
**both argument orders** and asserts the results agree, because order independence is
the property the merge exists for and a one-directional check would pass on an
order-dependent implementation.

### `signed_bytes`

Inputs: `input_device_id_hex`, `input_person_fingerprint_hex`, `input_version`,
`input_tombstone`. Outputs: `expected_message_hex`, `expected_prefix_ascii`, and
informational `informational_message_len_bytes` (80, constant). The three cases are a
live row, a tombstone, and a re-bind at a higher version on the same device.

### `verify`

Inputs: `verifier` (`person` or `owner`, i.e. whose public half to check against),
`input_device_id_hex`, the signer's three public-half parts, `input_entry` (the row).
Outputs: `expected_verified` (boolean) and informational
`informational_signed_bytes_hex`, which the runner cross-checks against
`device_binding_signed_bytes` so the message and the outcome cannot drift apart.

### `group_subkey`

Inputs: `input_group_key_hex` (32 B), `input_role` (`device-binding-key` or
`device-binding-sync-id`). Output: `expected_subkey_hex`. Two cases, one per role, and
both roles of the family must be pinned: the schema makes `input_role` an `enum`
rather than a `const` so a case can carry either, and the runner asserts the pinned
SET equals the family's two roles, so registering a third (or renaming one) fails
rather than shipping unvectored.

## Invariants the vectors pin

- **`MYU5` + both version bytes, checked before any decryption.** `MYU5` is its own
  blob family; a v0.1-only reader must refuse it.
- **Header byte layout and nonce offset.** `[6..30]` is the supplied nonce,
  `[30..34]` the big-endian `ciphertext_len`, `[34..]` ciphertext + 32-byte CTX tag.
- **The AAD binds the epoch.** Two blobs identical but for `group_key_epoch` MUST NOT
  open under each other's key: that is the whole point of the last AAD segment, and
  the `decrypt-fail-stale-epoch` case is what holds it.
- **The empty map round-trips** to 68 bytes and decodes back to empty.
- **Canonical bytes are the only accepted bytes.** `not_canonical` is a refusal, not a
  normalization.
- **The message class is fixed-width** at 80 bytes, so the signed bytes have exactly
  one encoding.
- **Signatures are not pinned; outcomes are.**
- **The merge is commutative**, checked in both argument orders rather than asserted.
- **The refusal is symmetric**: a same-version two-people pair fails in both orders,
  so the one case that cannot be resolved cannot be resolved *differently* by the two
  peers either.
- **Neither GROUP-tier role has a root-tier counterpart**, checked against
  `SubkeyRole::ALL` rather than asserted in prose.

## How to consume

A port dispatches on `case_kind`:

- **`encrypt`**: derive `device_binding_key` from `input_group_key_hex` via
  `GroupRole::DeviceBindingKey`, invoke the port's `encode_device_binding_blob_raw`
  equivalent with the supplied nonce, epoch, account hash and map, hex-encode, compare
  byte-equal to `expected_blob_hex`.
- **`decrypt_fail`**: invoke the decoder, assert the *named* failure. For the four
  `cannot_decrypt` cases the port MUST NOT distinguish which arm fired.
- **`parse_fail`** / **`encode_fail`**: assert the specific error named by
  `expected_error`.
- **`merge`**: merge `input_map_a` into `input_map_b` AND `input_map_b` into
  `input_map_a`; assert both results are equal to each other, and then equal to
  `expected_merged_map` or failing with the named `conflicting_binding` error.
- **`signed_bytes`**: rebuild the message with the port's own assembly, compare
  byte-equal.
- **`verify`**: build the signer's public half from the three published parts, verify
  the row's signature over the message, compare to `expected_verified`. For
  `verifier: "owner"` use `verify_device_binding_removal`; for `"person"` use
  `verify_device_binding`.
- **`group_subkey`**: derive the GROUP-tier subkey for `input_role` from
  `input_group_key_hex`, compare byte-equal.

Any mismatch is a stop-the-world bug (per repo `README.md` "Cross-implementation
discipline").

## Regeneration

```bash
cargo test -p protocol --lib emit_device_binding_vectors_for_paste -- --ignored --nocapture
```

The generator lives in `protocol/src/device_binding_blob.rs` and is `#[ignore]`-gated,
so it never runs in a default `cargo test`. It self-asserts every expectation at mint
(header facts, canonical round-trip, the epoch refusal on both sides, the
public-half round-trip, the merge's order independence) and prints the whole file.

**The committed file IS reproducible, verified rather than assumed**: two independent
runs of the command above were diffed and are identical, and the emitted document was
then diffed against the committed `cases.json` case by case, field by field, with zero
content differences. The committed file is the emitter's stdout pasted verbatim, so it
carries one trailing newline the emitter does not print: compare it parsed, or strip that
byte first, rather than treating raw byte equality as the test. That holds even for the
cases carrying real hybrid signatures, and the reason is worth stating because it is easy
to get backwards: `PersonKeyPair::sign`'s ML-DSA-65 half is genuinely randomized
(`sign_randomized`), but the generator drives it with a **fixed-seed `SeededRng`**, so the
signature bytes are a function of the seed and the message and nothing else. A future
emitter that reached for the OS CSPRNG would break this property silently, so the
reproducible-diff check is the thing to re-run after touching the generator.

Diff the emitted ids and expected bytes against the committed file case-by-case rather
than comparing counts: editing an emitter can drop one case and mint another in its
place while `cases.len()` stays green.

Runner: `device_binding_vectors` (`protocol/tests/vectors.rs`). The category also has
a row in that file's `mod embedded` table, which is what the wasm/browser lane reads.

## What this does NOT cover

- **Group rotation.** Re-sealing this blob under the new epoch, and queueing the
  retired epoch's blob-id for server reclaim, are the binding store's obligation in
  the main repo, not a vector gap: the vectors pin the epoch binding that makes the
  re-seal meaningful, and the `device-binding-sync-id` derivation that makes the
  reclaim addressable. What no case here can show is that the two halves are done
  in the right ORDER on removal.
- **The `getrandom` convenience wrapper** (`encode_device_binding_blob`): every
  family vectors only its deterministic `_raw` sibling.
- **Client-layer fallback discipline** (absent file / truncated / AEAD failure /
  unsupported format collapsing to "no bindings yet", via
  `decode_device_bindings_or_empty`), wrapper behaviour, verified by `myne-core`
  integration tests.
- **The roster.** Which `person_fingerprint` is a real member's, and where a
  `device_id` resolves to a device record, is the membership feature's surface; this
  family stores bindings.
