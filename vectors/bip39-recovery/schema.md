# `bip39-recovery` vector schema

Pins the **end-to-end recovery composition** — `protocol::phrase::normalize_user_phrase` (§ 7.4) → `protocol::derive_recovery_wrap_key_from_normalized` (§ 3.1) — for the Myne recovery flow (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)).

**Reference implementation:** `protocol/src/phrase.rs::normalize_user_phrase` + `protocol/src/kdf.rs::derive_recovery_wrap_key_from_normalized`.
**Spec:** `docs/spec/protocol.md` § 3.1 (recovery KDF) + § 7.4 (phrase normalization).

## What this category adds over `phrase_normalize` + `argon2id`

The two halves are already pinned individually: `phrase_normalize` pins `normalize_user_phrase`, and `argon2id` pins `derive_recovery_wrap_key_from_normalized` over already-normalized inputs. This category pins the **composition** — and one property the halves cannot pin in isolation:

> A raw, un-normalized user phrase is fed through `normalize_user_phrase` **and then directly into Argon2id** (NOT through BIP-39's PBKDF2 seed step). The NFKD normalization therefore happens **before** the KDF. Two raw phrases that differ only in case, whitespace, or NFKD-foldable presentation normalize to the same string and so **MUST derive the same `recovery_wrap_key`.**

A port that skips the NFKD pass (or applies it after the KDF, or in the wrong order) feeds different bytes to Argon2id and derives a different key — a bug the individual categories let slip but this one catches.

## The fixture: same 24 words, typed three ways

All three cases carry the canonical BIP-39 all-zero-entropy 24-word phrase (`abandon` ×23 + the checksum word `art`), the same `salt_random`, and the same `account_number_hash`. They differ ONLY in how the phrase is typed:

| `vector_id` | Variation exercised |
|---|---|
| `canonical-24w` | Already-normalized form (lowercase, single ASCII spaces, no padding) — `normalize_user_phrase` is a fixed point. Baseline key. |
| `messy-case-whitespace-24w` | Fully UPPERCASED, leading/trailing padding, double-space + tab between every word → exercises ASCII-lowercase + whitespace-coalesce + trim. |
| `nfkd-fullwidth-24w` | `art` typed in full-width Latin (U+FF41 U+FF52 U+FF54); two separators are a no-break space (U+00A0) and an ideographic space (U+3000) → exercises NFKD letter + whitespace folding. |

`informational_normalized_phrase_utf8_hex` is **identical** across all three, and `expected_recovery_wrap_key_hex` is **identical** across all three. That equality IS the contract.

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level) and [`schema/v0.1/bip39-recovery.json`](../../schema/v0.1/bip39-recovery.json) (per-case shape). Single case shape (no `case_kind`). Each case:

- `input_raw_phrase_utf8_hex` — the raw user-typed phrase, UTF-8 bytes (hex). The only per-case-varying input.
- `input_salt_random_hex` (16 B) + `input_account_number_hash_hex` (16 B) — recovery salt-domain inputs.
- `input_kdf_params` — `{ m_kib, t, p, argon2_version_hex }`; canonical Phase 1 = `{262144, 3, 2, "13"}`.
- `informational_normalized_phrase_utf8_hex` — the `normalize_user_phrase` output fed to Argon2id (auditable; identical across the set).
- `expected_recovery_wrap_key_hex` — the 32-byte `recovery_wrap_key` (identical across the set).

## How to consume

For each case, a port:

1. Hex-decodes `input_raw_phrase_utf8_hex` and interprets it as UTF-8.
2. Runs its `normalize_user_phrase` equivalent (NFKD → ASCII-lowercase → trim → whitespace-coalesce to single U+0020).
3. SHOULD assert the normalized bytes equal `informational_normalized_phrase_utf8_hex` (isolates a normalization bug from a KDF bug).
4. Feeds the normalized string **directly** to its `derive_recovery_wrap_key_from_normalized` equivalent with `salt_random`, `account_number_hash`, and `kdf_params` — **no PBKDF2 / BIP-39 seed step**.
5. Compares the 32-byte output byte-equal to `expected_recovery_wrap_key_hex`. Any divergence is a stop-the-world bug.

## Runtime

Argon2id at canonical params is **~3 s per case** (~9 s for the 3-case set) and allocates 256 MiB per derivation. Like `argon2id`, this category is **default-on** in `cargo test` (only the regenerator is `#[ignore]`-gated). Run in isolation with `cargo test -p protocol --test vectors bip39_recovery_vectors`.

## Invariants the vectors pin

- **NFKD-before-Argon2id.** The full-width / NBSP / ideographic-space case derives the same key as the canonical ASCII case — only possible if NFKD runs before the KDF.
- **Case + whitespace insensitivity.** Uppercase + irregular whitespace derive the same key — `normalize_user_phrase` folds them before derivation.
- **Direct-to-Argon2id (no PBKDF2).** The expected key is `Argon2id(normalized_phrase, salt)`, NOT `Argon2id(BIP39_seed(phrase), salt)`. A port that runs the standard BIP-39 PBKDF2 seed step first will derive a different key and fail every case.

## What this does NOT cover

- **BIP-39 checksum validation.** `derive_recovery_wrap_key_from_normalized` does not parse or validate the wordlist/checksum — it normalizes and derives. An invalid phrase produces a normalized string that simply fails AEAD-tag verification later at the `recovery_wrap` unwrap stage (non-distinguishing error per recovery-flow § 3.4). Wordlist-membership is out of scope here.
- **`RecoveryPhrase::join_normalized`.** The generation-side canonicalizer (for phrases Myne itself generated, already lowercase) is a separate path; this category pins the user-input path (`normalize_user_phrase`).
- **Non-canonical KDF params.** Only the canonical Phase 1 params are pinned. Param-block byte serialization is covered by `argon2id` + `vault-meta`.

## Regeneration

```bash
cargo test -p protocol --lib emit_bip39_recovery_vectors_for_paste -- --ignored   # ~9s
cp /tmp/myne_bip39_recovery_vectors.json vectors/bip39-recovery/v0.1/cases.json
```

The generator lives in `protocol/src/kdf.rs::tests::emit_bip39_recovery_vectors_for_paste`. It asserts in-process that all three raw inputs normalize identically and derive the same key before emitting — so a regeneration that breaks the composition fails loudly rather than shipping a contradictory vector set.
