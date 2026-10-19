# `dictionary-merge` vector schema

Pins the canonical `0x02` `vault.dictionary` payload and the `sync-protocol.md` § 10.2.1 merge of two such payloads, with a **per-word removal clock**. Reference implementation: `core/src/plugins/sync/dictionary_payload.rs` (the merge) and `protocol/src/dictionary_blob.rs` (the codec that wraps it). Spec: `docs/spec/sync-protocol.md` § 10.2.1 plus `docs/spec/protocol.md` § 9.13.2. Decision: `design notes/protocol/0710` (myne-md/myne#640).

This category exists because `vault.dictionary` did not sync at all before that decision: there was no sync-id derivation, no subkey role and no phase in the engine, so no peer ever held a copy and no merge ever ran. The clockless set union the spec described was therefore unreachable rather than wrong in the field. Enabling sync without clocks would have created the defect, so the removal clock ships in the first synced generation.

## The payload, and what is canonical

The `0x02` inner plaintext is a postcard-serialized struct holding the **added-words set**, the **ignore set**, and a **clock map**. Both word sets are NFC-normalized, sorted and de-duplicated (case folding is the LOOKUP layer's job; the codec stores exact NFC bytes); the clock map is sorted by key. A port that emits the same sets in a different order is byte-different and fails here, which is the point: these vectors pin the bytes, not only the set semantics.

## The merge (spec § 10.2.1)

`merge(local, remote)` is a union over both word sets in each set family, decided **per entry** by that entry's clock:

| Local | Remote | Result |
|---|---|---|
| present at clock `a` | absent, with clock `b` | kept when `a >= b`, dropped when `b > a` |
| absent, with clock `a` | present at clock `b` | kept when `b >= a`, dropped when `a > b` |
| present on both sides | present on both sides | kept, at the greater clock |
| absent, with a clock | absent, with a clock | the **greater clock is retained** even though neither side lists the word |

Three consequences are load-bearing and each is pinned by a case rather than asserted in prose:

1. **A retained tombstone is the whole point of the clock.** The fourth row keeps an orphan clock, because dropping it lets a third device that still holds the word at an older clock resurrect what two other devices already deleted.
2. **An equal clock keeps the entry.** An entry that survives one extra round can be deleted again by a later write; one dropped by a collision cannot be recovered, because the clock that would express the deletion is the one that was discarded.
3. **A removal clock is per word, not per set.** A per-set clock cannot represent an addition on one device and a removal on another within the same set, which is the ordinary case rather than an exotic one.

The single-sided decision is the same function the `vault.prefs` set fields use (`prefs_merge::single_sided_clock_rule`, myne-md/myne#613), not a second copy of it, so a divergence between the two families is a bug in one shared implementation rather than a drift between two.

## Mixed fleet

`dict_format_version 0x01` carries no clock map, so a peer that can only write `0x01` cannot express a removal. A dictionary removal therefore does not converge until every enrolled device is upgraded: the legacy peer's unchanged copy is unioned back in and the word returns. `0x01` blobs remain decodable and are read as words at clock `0`; a `0x01`-only decoder refuses a `0x02` blob with `DictFormatUnsupported` and falls back to the empty dictionary (`protocol.md` § 9.13.4), which is why an old client degrades to "no personal dictionary" rather than to a wrong one.

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/dictionary-merge.json`](../../schema/v0.1/dictionary-merge.json) (per-case shape; single `case_kind`, no discriminator). Run the repository's documented `ajv-cli validate` command (see the root `README.md`) to verify `cases.json` against the canonical schema. The sub-schema is authoritative; this document covers only the domain-specific semantics.

Each case pins one `(input_local_hex, input_remote_hex)` pair to `expected_merged_hex`, and every one of those three fields is a **canonical encoded payload**, not a decoded object. A port decodes both inputs, merges, re-encodes canonically, and compares hex byte-equal.

## How to consume

A port runs each case as: hex-decode `input_local_hex` and `input_remote_hex`, decode each as a `0x02` payload, call its own merge equivalent, encode the result canonically, hex-encode, and compare byte-equal to `expected_merged_hex`. Because the expected value is canonical bytes, a port whose merge is right and whose encoder is not still fails, which is deliberate: the bytes are what two implementations have to agree on.

## Invariants the vectors pin

The five cases cover, in order: a one-sided addition; a removal that converges because the absent side's clock is strictly greater; a retained tombstone that beats a third device's live entry; a re-add that beats its own tombstone; and an equal clock that keeps the entry. Three plausible wrong rules each fail a case **by name**, so a reviewer can confirm the vectors bite by implementing the wrong rule and watching which case goes red:

- **"A clock whose word is in neither input is dropped"** (the myne-md/myne#613 review finding, one family over) fails `merge-retained-tombstone-beats-third-device`.
- **"A single-sided entry always survives"** (the original clockless union) fails `merge-removal-converges`.
- **"Remove always wins at an equal clock"** (imported from the themes tombstone rule, § 10.2.2) fails `merge-equal-clock-keeps-the-entry`.
