# `password-policy` vector schema

Pins `protocol::validate_min_length` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)) — the master-password **minimum-length floor** enforced backend-side before any Argon2id work begins.

**Reference implementation:** `protocol/src/password_policy.rs::{validate_min_length, MIN_PASSWORD_LEN}`.
**Contract:** `design notes` 2026-05-06 "Auth-pipeline backend `MIN_PASSWORD_LEN` floor; project-wide retroactive".

## What this category is — and is not

This is a **boolean accept/reject conformance** category, not a byte-output category. Unlike every other category in this repo (which pins exact output bytes), `password-policy` pins a binary decision: does a candidate password clear the floor? The cross-implementation risk it guards is specific and real:

- `MIN_PASSWORD_LEN = 12` counts **Unicode scalar values** (`str::chars().count()`), **NOT** UTF-8 byte length.
- An independent client that gates on byte length instead of USV count diverges exactly on multibyte input: `"á".repeat(12)` is 12 USVs (accept) but 24 bytes; `"á".repeat(11)` is 11 USVs (reject) but 22 bytes. A byte-length gate accepts both — silently admitting a sub-floor password.

The floor is a **structural backstop against frontend bypass** (DevTools / hand-crafted IPC), not an entropy guarantee — `"a".repeat(12)` clears it and is still trivially weak. Entropy gating (zxcvbn) is deliberately frontend-only and out of scope here. The frontend `PasswordForm` component mirrors `MIN_PASSWORD_LEN` via a TS hand-constant; this category is the canonical drift-check for any port (including that TS mirror) that re-implements the floor.

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/password-policy.json`](../../schema/v0.1/password-policy.json) (per-case shape). The category has a **single case shape**; the outcome is carried by `expected_kind` (no `case_kind` discriminator). Run the repository's documented `ajv-cli validate` command (see the root `README.md`) to verify `cases.json` against the canonical schema.

Each case:

- `vector_id` — stable `password-policy/v0.1/<slug>` identifier.
- `expected_kind` — `"accept"` (`Ok(())`) or `"reject_too_short"` (`Err(ProtocolError::PasswordTooShort)`). Sub-floor length is the only rejection cause at this layer.
- `input_password_utf8_hex` — the candidate password as UTF-8 bytes, hex-encoded so whitespace / control chars / multibyte scalars transport unambiguously. Empty allowed (zero bytes → reject).
- `informational_usv_count` — `str::chars().count()` of the decoded password; the value compared against the floor.
- `informational_byte_len` — UTF-8 byte length; differs from the USV count on multibyte cases.

## How to consume

A port runs each case:

1. Hex-decode `input_password_utf8_hex` and interpret as UTF-8.
2. Invoke the port's `validate_min_length` equivalent.
3. Assert the outcome matches `expected_kind` (`accept` ⇒ pass, `reject_too_short` ⇒ the port's PasswordTooShort-equivalent).
4. SHOULD additionally assert its own scalar-value count equals `informational_usv_count` and its byte length equals `informational_byte_len` — this is what catches a byte-length-vs-USV bug directly, rather than relying on the multibyte fixtures alone to surface it.

## Cases (7)

| `vector_id` | USVs | bytes | `expected_kind` |
|---|---|---|---|
| `accept-12-ascii` | 12 | 12 | accept |
| `accept-13-ascii` | 13 | 13 | accept |
| `reject-11-ascii` | 11 | 11 | reject_too_short |
| `reject-empty` | 0 | 0 | reject_too_short |
| `accept-12-precomposed-a-acute` | 12 | 24 | accept |
| `reject-11-precomposed-a-acute` | 11 | 22 | reject_too_short |
| `accept-13-mixed-script` | 13 | 21 | accept |

The two precomposed-`á` rows are the load-bearing pair: same byte counts on either side of the floor would mislead a byte-gating port, but the USV count is what decides.

## Invariants the vectors pin

- **The floor is 12 USVs.** 11 rejects, 12 accepts, 13 accepts (ASCII rows pin the boundary exactly).
- **USV counting, not byte counting.** The precomposed-`á` rows (12 USV / 24 byte accept; 11 USV / 22 byte reject) and the mixed-script row (13 USV / 21 byte, including a 4-byte astral scalar) make a byte-length gate observably wrong.
- **Empty rejects.** Zero-length is below the floor like any other sub-12 input — no special-casing.
- **Length is the sole cause.** Every reject is `reject_too_short`; there is no other rejection mode at this layer (no charset / entropy rules backend-side).

## What this does NOT cover

- **Entropy / strength gating (zxcvbn).** Frontend-only by design; `protocol` carries no entropy estimator. A 12-`a` password clears this floor and is intentionally still admitted here.
- **Grapheme clusters.** The count is over Unicode scalar values, which differs from user-perceived graphemes when combining marks are involved (e.g. `a` + U+0301 is 2 USVs). The floor is approximate by design — adequate for a structural backstop, not a typography-aware rule.
- **Normalization.** `validate_min_length` does not NFC/NFKD-normalize before counting; it counts the scalars as received. A decomposed sequence therefore counts more USVs than its precomposed form — by design at this layer.

## Regeneration

```bash
cargo test -p protocol --lib emit_password_policy_vectors_for_paste -- --ignored
cp /tmp/myne_password_policy_vectors.json vectors/password-policy/v0.1/cases.json
```

The generator lives in `protocol/src/password_policy.rs::tests::emit_password_policy_vectors_for_paste`. `#[ignore]`-gated; emits ready-to-copy JSON. Byte-level/decision conformance against the live Rust impl runs via `cargo test -p protocol --test vectors password_policy_vectors`.
