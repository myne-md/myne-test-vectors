# `mcp-write-request` — the external-MCP write queue

Pins `protocol::parse_mcp_write_request` and the canonical encoding of the
`mcp_write_requests` array embedded inside `vault.prefs`.

- **Reference implementation:** `protocol/src/prefs.rs` (main `myne` repo)
- **Spec anchor:** `protocol.md` §9.5.2 (`mcp_write_requests`, the queue beside `mcp_connections`)
- **Design:** `design notes/ai/0752-a-write-grant-is-a-named-allow-list-and-every-write-is-a-validated-intent-with-a-snapshot.md`
- **Runner:** `protocol/tests/vectors.rs::mcp_write_request_vectors` (`cargo test -p protocol mcp_write_request_vectors`)
- **Generator:** `cargo test -p protocol --lib emit_mcp_write_request_vectors_for_paste -- --ignored` → `/tmp/myne_mcp_write_request_cases.json`

## What this category pins

One entry says: **this connection tried to reach into this folder, at this time,
with this verb, and said this about it.** Two kinds of attempt queue here and they
move the grant in opposite directions: a `write` row asks to ADD one folder to the
write list, and a `read` row asks to REMOVE one folder from the exclusion set. It is a *proposal*, not a pending write.
Nothing has been applied; accepting the row extends that connection's write grant
by exactly the folder it names, and dismissing it drops the row without telling
the client.

So the queue sits in the one place where a reader can go wrong in both
directions:

- **a dropped row** hides a decision the person was owed, and the client is never
  told, so it silently queues again;
- **an invented row** spends an approval nobody gave. This is why the four
  required members are required rather than defaulted: a row that cannot be
  matched to a live grant, or cannot be described to the person deciding, is
  refused instead of shown.

`summary` is **CLIENT-AUTHORED USER TEXT**. It is rendered and never logged, it is
omitted from the encoding when empty, and no other member of this family is
free-form.

## The members

| member | type | why it is here |
|---|---|---|
| `connection` | string | the grant that would be widened. Resolved against `mcp_connections` at render time, never carried as authority |
| `folder` | string | the one folder to add, canonical. `""` is the vault root, and it is a folder key like any other |
| `kind` | string, optional | `read` or `write`. **Omitted when it is `write`**, so a row queued before the member existed keeps its meaning and re-encodes byte for byte. Any other token is refused: reading an unknown kind as a write would ask the person to widen the wrong half of the grant |
| `requested_at` | integer | Unix epoch seconds, stamped by the host. Orders the card so the newest proposal is the first one read |
| `summary` | string, optional | what the client said it wanted to write. Omitted when empty |
| `verb` | string | the tool it called (`note_create`, `note_append`, ...). Rendered as a sentence, never as a raw identifier |

## Why the write scope is an allow list beside a deny-list read scope

Deliberately asymmetric, and it is the reason this family exists at all:

- **read** is a `deny list` (`excluded_folders`): revoking an exclusion is a
  reversible act, so the failure of forgetting to exclude something is a visible
  one;
- **write** is an `allow list` (`write_folders`): a write into a folder nobody
  anticipated is the surprise worth preventing, and it cannot be undone by
  reading.

An empty `write_folders` therefore means **write nowhere**, which is the exact
inversion of an empty `excluded_folders` meaning **the whole vault**. Both are the
empty set in the bytes and they mean opposite things, so the two are never read by
the same code path.

## Reject set

An unknown member, a repeated member, a missing required member, and a member
whose type is wrong each fail the whole object. The reference decoder answers
"absent" rather than a typed error, so the assertion is *must not be read*: a
reader that resolves a malformed queue to an empty one has hidden every proposal
in it, and a reader that resolves it to a partial one has shown a proposal that
says less than the client wrote.
