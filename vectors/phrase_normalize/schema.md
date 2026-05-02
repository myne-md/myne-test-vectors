# `phrase_normalize` vector schema

Pins the byte mapping of `protocol::phrase::normalize_user_phrase` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)) so that any independent implementation produces bit-identical normalization output. The reference implementation lives at `protocol/src/phrase.rs`; the pipeline is mandated by `docs/spec/protocol.md` § 7.4.

## Pipeline (as fixed by `protocol.md` § 7.4)

For each input UTF-8 string:

1. **NFKD compatibility decomposition.** Folds full-width digits (U+FF10..U+FF19), full-width Latin letters (U+FF21..U+FF3A), no-break space (U+00A0), ideographic space (U+3000), ligatures (e.g. U+FB01 `ﬁ` → "fi"), and other compatibility-decomposable characters into their canonical forms.
2. **ASCII lowercase.** The BIP-39 English wordlist is purely ASCII; user-typed uppercase (or post-NFKD uppercase from full-width letters) lowercases to match. Unicode lowercase is not used — ASCII-only is sufficient and deterministic.
3. **Trim + whitespace coalesce.** Leading and trailing runs of Unicode whitespace are dropped. Internal runs of one or more Unicode whitespace characters between tokens collapse to a single ASCII space U+0020.

## File layout

```
vectors/phrase_normalize/
├── schema.md              ← this file
└── v0.1/
    └── cases.json         ← array of input → expected_output cases
```

## JSON shape

Top-level object:

| Field | Type | Description |
|---|---|---|
| `schema` | string | Always `"phrase_normalize"` for this directory. |
| `protocol_version` | string | `"v0.1"` for v0.1 vectors. |
| `spec_section` | string | Anchors the canonical pipeline definition (`"protocol.md §7.4"`). |
| `description` | string | Human-readable summary of what is pinned. |
| `cases` | array | One element per test case (see below). |

Each `cases[i]` element:

| Field | Type | Description |
|---|---|---|
| `vector_id` | string | Stable identifier (`"phrase_normalize/v0.1/<slug>"`). Immutable once shipped. |
| `description` | string | What the case verifies. |
| `input_utf8_hex` | string | Lowercase hex (no `0x` prefix) of the UTF-8 byte encoding of the input string. |
| `expected_output_utf8_hex` | string | Lowercase hex (no `0x` prefix) of the UTF-8 byte encoding of the expected normalized output string. |

## How to consume

A port runs each case as: decode `input_utf8_hex` to bytes, parse as UTF-8 string, pipe through the port's `normalize_user_phrase` equivalent, encode the output as UTF-8 bytes, hex-encode lowercase, compare byte-equal to `expected_output_utf8_hex`. Any non-equal result is a stop-the-world bug — either the port has drifted or the spec is wrong (per repo `README.md` "Cross-implementation discipline").

## Why hex, not JSON-native UTF-8 strings

JSON natively supports Unicode strings via `\uXXXX` escapes, but escape behavior varies subtly across language libraries (some normalize on parse, some collapse line endings, some treat surrogate pairs differently). Hex-encoding the raw UTF-8 bytes eliminates every parser-side transformation: the bytes that go in are exactly the bytes the reference implementation saw.

## What this does NOT cover

- Wordlist validation. `normalize_user_phrase` does not check whether the normalized words are in the BIP-39 English wordlist; that is intentional per `recovery-flow.md` § 3.2 (no per-cell wordlist-membership feedback). Invalid phrases produce a normalized string that fails AEAD-tag verification downstream.
- The full recovery flow (phrase → Argon2id → AEAD-decrypt). That belongs to a future `vectors/bip39-recovery/` directory which composes `phrase_normalize` with `argon2id` + `xchacha20-poly1305-ctx`.
