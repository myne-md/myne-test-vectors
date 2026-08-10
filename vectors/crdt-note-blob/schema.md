# `crdt-note-blob` — NC05 next-major (v0.2) CRDT note-type codec

Cross-implementation vectors for the `NC05` blob codec (`protocol.md` §10.7,
automerge-sync-migration #418). `NC05` is the next-major (v0.2) note-type: the
same §9.2 three-level envelope as the `MY01` per-note blob, with exactly three
changes that domain-separate v0.2 from v0.1 (§10.5):

1. the 4-byte magic is `NC05` (not `MY01`);
2. the header `protocol_version` byte is `0x02` (not `0x01`);
3. both AEAD tags bind the v0.2 AAD prefix `"myne.v0.2|"` + version `0x02`
   (via `aad_nc05_item_key_wrap` / `aad_nc05_content_encrypt`), so an `NC05`
   tag never verifies under a v0.1 AAD and vice versa.

The inner plaintext is **opaque** — the note's Automerge `save()` bytes — and the
codec never parses it (like `vault.search` / `vault.workspace`). `content_type`
stays the hardcoded `"text/markdown"`; the magic, not the AAD, is the codec
discriminator.

Reference implementation: `protocol/src/crdt_note_blob.rs`
(`encode_crdt_note_blob_raw` / `decode_crdt_note_blob` + the two `aad_nc05_*`
builders). Generator: `emit_crdt_note_blob_vectors_for_paste` in that module.

## Version note

This family lives under `v0.2/` (not `v0.1/`) because the format IS the v0.2
next-major note-type — the first vector family for a next-major format. The
native + browser runners read `crdt-note-blob/v0.2/cases.json`.

## Case kinds

| `case_kind` | What it pins |
|---|---|
| `encrypt` | full byte output of `encode_crdt_note_blob_raw` under fixed inputs |
| `decrypt_ok` | round-trip: the blob decodes back to its opaque payload |
| `decrypt_aead_fail` | non-distinguishing AEAD failure — the **v0.1-AAD domain-separation** case (an `NC05` wrap sealed under the v0.1 AAD must not open) + a wrong-`items_key` cross-vault case |
| `parse_fail` | structural pre-decryption rejects: `MY01` magic, v0.1 version byte, non-zero reserved, and strict-trailing (§10.3.1) |
| `aad_construction` | canonical bytes of `aad_nc05_item_key_wrap` + `aad_nc05_content_encrypt` |

Non-distinguishing `decrypt_aead_fail` / `parse_fail` fixtures carry the **real**
`items_key`: header bytes are not AAD-bound, so a port that skips a structural
check decodes `Ok` and the runner catches it — a dummy key would mask the bug as
a coincidental AEAD failure.
