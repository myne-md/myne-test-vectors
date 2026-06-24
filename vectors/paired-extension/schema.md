# `paired-extension` — pairing-token registry decoder

Pins `protocol::paired_extension::parse_paired_extensions_array` — the
canonical-JSON decoder for the `paired_extensions` array embedded inside
`vault.prefs`.

- **Reference implementation:** `protocol/src/paired_extension.rs` (main `myne` repo)
- **Spec anchor:** `protocol.md` §9.5.2 (browser-clipper persisted pairing-token registry)
- **Runner:** `protocol/tests/vectors.rs::paired_extension_vectors` (`cargo test -p protocol paired_extension_vectors`)
- **Generator:** `cargo test -p protocol --lib emit_paired_extension_vectors_for_paste -- --ignored` → `/tmp/myne_paired_extension_vectors.json`

## What this category pins

The browser clipper's persisted pairing registry is a JSON **array** of records,
each binding an `extension_id` to a bearer `token_value`, a `revoked` flag, and
the additive optional `paired_at` issue timestamp. This category pins the
**decode** direction: input bytes → `(Vec<PairedExtension>, consumed)`.

The bytes are decoded only AFTER the surrounding `vault.prefs` AEAD has verified.
Like `saved-view`, this is deterministic cross-client behavior — not a codec,
not cryptographic in itself — but load-bearing for cross-implementation
agreement.

One subtlety the vectors exercise: `token_value` is `#[serde(skip_serializing)]`
so the bearer secret never crosses the WebView IPC boundary, **but** the
hand-rolled on-disk canonical codec writes it explicitly. The decode→re-encode
projection therefore round-trips `token_value` — it is part of the on-disk
contract even though it is hidden from the IPC view.

## Case shape

```
cases[]:
  vector_id    "paired-extension/v0.1/<slug>"
  case_kind    "parse_ok" | "parse_fail"
```

### `parse_ok`

| field | meaning |
|---|---|
| `input_bytes_utf8_hex` | the input bytes (a JSON array, optionally with trailing bytes) |
| `expected_consumed_len` | bytes the decoder MUST report consumed — the array length only |
| `expected_canonical_utf8_hex` | the decoded set re-encoded via `canonical_encode_paired_extensions` |

The canonical projection captures:

- **NFC folding** of strings (`nfc-fold-extension-id`).
- **Per-record key order** `extension_id` < `paired_at` < `revoked` < `token_value`.
- **Additive optional `paired_at`**: emitted only when present, so a pre-feature
  record is byte-identical to the legacy encoding (`roundtrip-no-paired-at` vs
  `roundtrip-with-paired-at`).
- **`(extension_id, token_value)`-sort** of the array (`multi-sorted`).
- **Unknown-field drop** (forward-compat, `unknown-field-tolerated`).

### `parse_fail`

| field | meaning |
|---|---|
| `input_bytes_utf8_hex` | the malformed/invalid input bytes |
| `expected_error` | one of `malformed` / `missing_field` / `wrong_type` |

(No `unrepresentable` — the record schema carries no closed enum value the way a
saved view's `view_type` / filter `kind` do.)

## How to consume

For each `parse_ok` case: hex-decode `input_bytes_utf8_hex`, run your decoder,
assert it consumed exactly `expected_consumed_len` bytes, re-encode the decoded
set canonically, and assert byte-equality with `expected_canonical_utf8_hex`.
For each `parse_fail` case: assert your decoder rejects with the named error.

## Invariants the vectors pin

- The **cursor contract** (`consumes-only-the-array-trailing`).
- The **additive-optional byte-identity** of `paired_at` — a forward/backward
  compatibility guarantee.
- The on-disk presence of `token_value` despite its IPC serde-skip.
- The JSON-float ban (`exponent-float-banned`).

## What this does NOT cover

The host-side pairing handshake, constant-time token comparison, and revocation
gate (`src-tauri/src/clipper/token.rs`) are not vectored — the token is not a
key and carries no vault material. The `vault.prefs` AEAD framing that wraps this
array is the `vault-prefs` category's job.
