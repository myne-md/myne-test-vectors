# `rename-journal` — transactional-rename crash recovery

Pins `note_txn::recover_pending_rename` — the §9.12 rename-journal crash-recovery
state machine — as an **fs-state conformance fixture**.

- **Reference implementation:** `src-tauri/src/note_txn.rs` (main `myne` repo)
- **Spec anchor:** `protocol.md` §9.12 (transactional multi-note rename)
- **Runner:** `src-tauri/src/note_txn.rs::rename_journal_vectors` (`cargo test -p myne-desktop --lib rename_journal_vectors`)
- **Generator:** `cargo test -p myne-desktop --lib emit_rename_journal_vectors_for_paste -- --ignored` → `/tmp/myne_rename_journal_vectors.json`

This is the second std/fs-bound category whose generator + runner live in the
main repo's `src-tauri/` (not the `protocol/` crate), after `backup-container`.

## Why a snapshot, not a byte vector

The rename journal is a **transactional procedure**, not an on-disk blob codec.
Its persisted state is minimal and content-free: an empty `.rename-journal`
marker file whose mere **presence** is the durability fence, plus `*.myn.tmp`
staged blobs. POSIX `readdir` is unordered, so there is no byte sequence to pin.

So each case pins a **directory snapshot** — a name-sorted set of
`{name, content_hex}` — before and after recovery:

```
input_dir_files   → recover_pending_rename → expected_dir_files
```

## The state machine pinned

`recover_pending_rename` reads the marker, then acts on every staged `*.myn.tmp`:

- **Marker present → roll FORWARD.** The marker proved every staged blob is
  durable, so each `<uuid>.myn.tmp` is atomically renamed to `<uuid>.myn`
  (replacing any existing committed file), and the marker is cleared.
- **Marker absent → roll BACK.** The crash happened before the durability fence,
  so each `<uuid>.myn.tmp` is deleted. No committed state changes.
- Committed `*.myn` files (not staged, not the marker) are always untouched.

Recovery is **idempotent**: a second run is a fixed point (after roll-forward or
roll-back the directory has no marker and no `.tmp`, so nothing more happens).

## Case shape

```
cases[]:
  vector_id           "rename-journal/v0.1/<slug>"
  case_kind           "roll_forward" | "roll_back" | "noop"
  input_dir_files     [ { name, content_hex } ]   (name-sorted set)
  expected_dir_files  [ { name, content_hex } ]   (name-sorted set)
```

`.rename-journal` is the marker (empty `content_hex`); `*.myn.tmp` are staged
blobs; `*.myn` are committed blobs.

## How to consume

Materialize `input_dir_files` in a fresh directory, run your recovery routine,
snapshot the resulting directory as a name-sorted `{name, content_hex}` set, and
assert it equals `expected_dir_files`. Re-run recovery and assert the snapshot is
unchanged (idempotency).

## What this does NOT cover

The forward path (`write_batch_transactional` — stage → mark → commit → clear)
is exercised by the in-repo unit tests with a fault-injection seam, not by these
vectors. Durability primitives (`fsync` ordering, atomic `rename(2)`) are
platform behavior the OS guarantees; the vectors pin the recovery **decision**
(forward vs back vs noop), not the syscall sequence.
