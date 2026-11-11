# `mcp-connection` — external-MCP connection grant decoder

Pins `protocol::parse_mcp_connection_grant` — the canonical-JSON decoder for one
record of the `mcp_connections` map embedded inside `vault.prefs`.

- **Reference implementation:** `protocol/src/prefs.rs` (main `myne` repo)
- **Spec anchor:** `protocol.md` §9.5.2 (external MCP connection grants, `mcp_connections`)
- **Runner:** `protocol/tests/vectors.rs::mcp_connection_vectors` (`cargo test -p protocol mcp_connection_vectors`)
- **Generator:** `cargo test -p protocol --lib emit_mcp_connection_vectors_for_paste -- --ignored` → `/tmp/myne_mcp_connection_vectors.json`

## What this category pins

`mcp_connections` is a map from a declared connection name to one grant object.
The grant records which folders an external MCP client (`myne-mcp`) may **NOT**
read, plus when it was approved and when it last reached the door. The bytes are
decoded only AFTER the surrounding `vault.prefs` AEAD has verified, so this is
deterministic cross-client behaviour rather than a codec or a cryptographic
operation — but it is load-bearing for consent, because getting it wrong is
invisible.

## The members

| member | type | why it is here |
|---|---|---|
| `approved_at` | integer | Unix epoch seconds of the approval, stamped from the host's own clock. Required |
| `excluded_folders` | array | the deny list under `model: exclude`. Empty means the WHOLE vault |
| `folders` | array | the retired allow list under `model: folders`. Empty means NO folder |
| `last_used_at` | integer, optional | epoch seconds of the last call that reached the host. Omitted when never used |
| `model` | string, optional | the discriminator. Emitted only for `exclude`, so an older blob re-encodes byte for byte |
| `write_folders` | array, optional | the write ALLOW list, present under `exclude` only. Empty or absent means write nowhere |
| `disabled_verbs` | array of strings, optional | the catalog verbs this connection may NOT use, as a canonical **set** (sorted, exact-deduplicated). **Omitted when empty**, which means every verb is available: a grant written before the member existed must keep its full set, because the other reading would turn an upgrade into a revocation. A reader MUST refuse it beside the retired `folders` model, which has no verb vocabulary to subtract from |

## The model discriminator, and why shape-inference is not allowed

Two models have shipped:

| `model` member | array | meaning |
|---|---|---|
| absent, or `"folders"` | `folders` | **retired allow list**: the connection's whole reach, frozen at approval |
| `"exclude"` | `excluded_folders` | **this build's deny list**: everything is readable except these folders |

Both are arrays of folder paths, so a reader that guesses from the shape inverts
consent: a stored `["Private/"]` flips from "only `Private/`" to "everything
except `Private/`", and nothing in the bytes looks wrong. So the discriminator is
stored, `model` is emitted only for `exclude` (which makes a blob written by the
earlier build re-encode byte for byte), and a reader MUST refuse a grant whose
`model` and arrays disagree rather than resolve it by precedence.

A reader that meets a `folders` grant MUST still accept it, MUST NOT serve it as
a deny list, and MUST refuse a call made under one (`reapproval_required` on the
wire), because widening a consent as a side effect of an upgrade is the one
direction a security model may not move on its own.

## Case shape

```
cases[]:
  vector_id    "mcp-connection/v0.1/<slug>"
  case_kind    "parse_ok" | "parse_reject"
```

### `parse_ok`

| field | meaning |
|---|---|
| `input_bytes_utf8_hex` | the input bytes (a JSON object, optionally with trailing bytes) |
| `expected_consumed_len` | bytes the decoder MUST report consumed — the object length only |
| `expected_canonical_utf8_hex` | the decoded grant re-encoded via `canonical_encode_mcp_connection_grant` |

The canonical projection captures:

- **Member order** `approved_at` < `excluded_folders` < `folders` < `last_used_at` < `model`.
- **The omission rules that make re-encoding a fixed point**: `model` only for
  `exclude`; `excluded_folders` only when non-empty (an empty exclusion set is a
  complete grant meaning the whole vault, and costs no array); `folders` always,
  even empty, because under the legacy model "nothing was captured" and "nothing
  is readable" are different statements.
- **Folder sets are sets**: NFC folding, exact deduplication, lexicographic order.
  Case is PRESERVED in the stored set; the resolver lowercases on read, so a
  differently-cased sibling stays two entries at rest and one key in effect.

### `parse_reject`

| field | meaning |
|---|---|
| `input_bytes_utf8_hex` | the input bytes that MUST be refused |

The rejection set is the half a round-trip case cannot express, and it is where a
port drifts. A reader MUST refuse: both arrays in one object; an exclusion list
beside no `model`; the legacy model with no array; an unknown `model` token; an
unknown member; a repeated member; an object with no `approved_at`; and a write
list beside the retired allow-list model. The reference decoder collapses all of
these to `None`, so this family pins "must be refused" rather than a variant name.

`reject-unknown-member` uses a member **no build writes**. It used to name
`write_folders`, which was right while the write grant was a future extension, and
it stopped being a test of anything the moment a build learnt that member: the
fixture would then be asserting that a reader refuses what it now accepts. The
example moved rather than the rule.

## The write scope, and why it is an allow list beside a deny-list read scope

`write_folders` names the folders the connection MAY write, and it is the one
member here whose empty case means the opposite of its sibling's:

| member | model | empty means |
|---|---|---|
| `excluded_folders` | deny list over read | **the whole vault** |
| `write_folders` | allow list over write | **write nowhere** |

The asymmetry is deliberate. Revoking a read exclusion is a reversible act, so a
deny list fails visibly: the person sees what was shared and takes it back. A
write cannot be taken back by reading, so the surprise worth preventing is a write
into a folder nobody named, and only an allow list prevents it.

Three consequences are pinned as cases rather than left to prose:

- `write_folders` sorts **after** `model`, so a grant written before this member
  existed re-encodes byte for byte;
- an explicitly empty list **re-encodes without the member**, and a reader must
  not read that absence as "everywhere";
- a write list beside the **retired** `folders` model is refused outright. That
  model's array is the connection's whole reach, the write grant was introduced
  with the deny list, and one approval with two incompatible readings is the shape
  this family exists to refuse.

A folder named for writing must also be one the connection may READ. That rule is
enforced where the grant is written (`core/src/plugins/prefs/commands.rs`) rather
than in the codec, because it is a rule about a pair of members and the codec sees
one object at a time; a grant that violates it is refused at approval, so it never
reaches the bytes this family describes.

## Cases

| vector | pins |
|---|---|
| `exclude-model-roundtrip` | the model this build writes, with a non-empty exclusion set |
| `exclude-empty-is-the-whole-vault` | empty exclusion set is a value, and costs no array |
| `legacy-folders-model-roundtrip` | a retired allow-list grant re-encodes byte for byte |
| `legacy-folders-empty-array` | the legacy array is required, even when empty |
| `last-used-at-emitted-when-present` | the optional usage stamp's position and omission |
| `folders-nfc-sorted-deduped` | NFC folding, exact dedup, lexicographic order, case preserved |
| `write-grant-rides-after-model` | the write allow list's position, and its set encoding |
| `write-grant-empty-list-is-omitted` | an empty write list means write nowhere and costs no bytes |
| `served-once-a-read-write-grant-records-it` | the usage stamp beside a write grant |
| `consumes-only-the-object-trailing` | the cursor-advance contract |
| `reject-both-arrays` | model/array disagreement |
| `reject-exclusion-beside-allow-list` | an exclusion list with no discriminator |
| `reject-allow-list-without-array` | the legacy shape missing its array |
| `reject-unknown-model-token` | a third model cannot be defaulted into either |
| `reject-unknown-member` | forward-compat does not extend to this object |
| `reject-write-list-on-the-retired-model` | one approval may not carry two readings |
| `reject-repeated-member` | two answers to one question |
| `reject-missing-approved-at` | a grant nobody can date is not an approval |
