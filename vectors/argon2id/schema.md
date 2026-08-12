# `argon2id` vector schema

Pins the byte mapping of `protocol::derive_password_wrap_key` + `protocol::derive_recovery_wrap_key_from_normalized` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)) under canonical Phase 1 Argon2id parameters. Each case fixes a `(password-or-phrase, salt_random, account_number_hash, KdfParams, path)` tuple and pins the resulting 32-byte AEAD wrap key.

**Reference implementation:** `protocol/src/kdf.rs::{derive_password_wrap_key, derive_recovery_wrap_key_from_normalized}`.
**Spec:** `docs/spec/protocol.md` § 3 (salt prefixes + KDF) + § 4 (KdfParams) + `design notes.md` 2026-04-26 KDF-calibration entry.

## Pipeline (as fixed by spec § 3.1)

For each `(input, salt_random, account_number_hash, params)` tuple:

1. **Build the salt** = `SHA-256(prefix || account_number_hash || salt_random)[:16]`, where the prefix is path-specific:
   - `path = password` → `b"myne.v0.1 argon2id-password|"` (28 ASCII bytes incl. trailing pipe)
   - `path = recovery` → `b"myne.v0.1 argon2id-recovery|"` (28 ASCII bytes incl. trailing pipe)
2. **Argon2id derive** with the given `KdfParams` (`m_kib`, `t`, `p`, `argon2_version`), the 16-byte salt, and the UTF-8 byte encoding of the input (`password.as_bytes()` for the password path; the already-NFKD-normalized `phrase_normalized.as_bytes()` for the recovery path). Output: 32-byte AEAD-key material wrapped in `Zeroizing` so the bytes are wiped on drop.

The two paths share Argon2id parameters but differ in the salt prefix (domain separation) and in whether the input has been NFKD-normalized upstream:
- **Password path**: Myne does NOT normalize passwords on the client side. The bytes the user types are the bytes fed to Argon2id. This is intentional — passwords are not BIP-39 phrases; spec § 3 doesn't normalize them.
- **Recovery path**: The caller MUST NFKD-normalize the phrase before invoking `derive_recovery_wrap_key_from_normalized` (per spec § 3.4 + spec § 7.4 `phrase_normalized` shape). The function is the "already normalized" entry point; the structured `RecoveryPhrase` constructor (`derive_recovery_wrap_key`) normalizes upstream.

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/argon2id.json`](../../schema/v0.1/argon2id.json) (per-case shape — single case_kind, `input_path` field dispatches password vs recovery at the runner). Run the repository's documented `ajv-cli validate` command (see the root `README.md`) to verify any `cases.json` against the canonical schema. The sub-schema is authoritative — this document covers only the domain-specific semantics.

Each case carries:
- `input_path`: enum `"password"` | `"recovery"` — selects which derive function the runner calls.
- `input_password_or_phrase_utf8_hex`: UTF-8 byte encoding of the password (for `path=password`) or the NFKD-normalized phrase (for `path=recovery`).
- `input_salt_random_hex`: 16-byte Argon2id salt entropy.
- `input_account_number_hash_hex`: 16-byte SHA-256-truncated account-number hash.
- `input_kdf_params`: `{ m_kib, t, p, argon2_version_hex }` — pinned to canonical v0.1 values via schema `const` constraints.
- `expected_wrap_key_hex`: 32-byte derived wrap key.

## How to consume

A port runs each case by:
1. Decoding all `input_*_hex` fields.
2. Parsing `input_kdf_params` into the port's KdfParams type.
3. Calling the path-appropriate function (`derive_password_wrap_key` for `path=password`; `derive_recovery_wrap_key_from_normalized` for `path=recovery`).
4. Comparing the returned 32-byte output byte-equal to `expected_wrap_key_hex`.

## Runtime cost

**~3 seconds per case** at canonical v0.1 parameters (m=256 MiB, t=3, p=2). With 7 cases the category adds ~21 seconds to a default `cargo test -p protocol --test vectors` run. This is the **only** category in the landed Lane D set with non-trivial runtime; all other categories run in under a second total.

The cost is accepted as the trade-off for cross-implementation byte-level verification at the deployed parameters. The vectors are NOT `#[ignore]`-gated — protocol-contract assertions warrant default execution. Only the regenerator (`emit_argon2id_wrap_vectors_for_paste`) is `#[ignore]`-gated, since regeneration is a deliberate act.

## Invariants the vectors pin

For the 7 cases in `cases.json` (password × {short, long, unicode}, recovery × {short, long, NFKD-unicode}, and the `quickunlock-pin` path):

- **Canonical Argon2id parameters across all cases.** `m_kib = 262_144` (256 MiB), `t = 3`, `p = 2`, `argon2_version = 0x13`. The schema's `const` constraints on each field structurally enforce this — a future regenerator that lands fast-test params accidentally would fail schema validation. Future v0.1.x mobile-profile vectors (lower memory cost) would land in a separate sub-directory or a successor schema version.
- **Two distinct salt prefixes domain-separate the two paths.** Password derivations use `"myne.v0.1 argon2id-password|"`; recovery derivations use `"myne.v0.1 argon2id-recovery|"`. Same `(salt_random, account_number_hash, password_bytes)` tuple produces DIFFERENT wrap keys depending on path. The six password/recovery cases (password-short + recovery-short, password-long + recovery-long, password-unicode + recovery-nfkd-unicode) deliberately use the same `salt_random` + `account_number_hash` so any cross-path drift surfaces as differing wrap keys for the same other inputs.

- **A third path: the quick-unlock PIN.** The seventh case pins `derive_quick_unlock_pin_wrap_key` (§9.14.2) — the PIN run through Argon2id under its own salt prefix `"myne.v0.1 argon2id-quickunlock-pin|"`, with `input_salt_random_hex` carrying the 16-byte `pin_salt`. It is a **third** domain, not a variant of the password path: the key it produces wraps the device-local third `root_key` copy in `vault.quickunlock` (`MYQ1`), so a port that reused the password prefix here would produce a credential that unlocks under the wrong secret. The runner pins all three derivation paths through a coverage guard rather than a case count.
- **NFKD normalization byte-output is observable.** The `recovery-nfkd-unicode` case feeds 11 UTF-8 bytes representing the NFKD decomposition of `"şifre-ñ"` (`s` + COMBINING CEDILLA + `ifre-` + `n` + COMBINING TILDE). An implementation that incorrectly skips the NFKD step would feed the 9 NFC bytes (`ş` as 0xC5 0x9F instead of `s` + 0xCC 0xA7) and produce a different wrap key. Cross-impl ports MUST honor the NFKD pre-normalization contract for recovery; the case demonstrates the byte-level consequences.
- **Raw unicode in passwords (no Myne-side normalization).** The `password-unicode` case feeds the 9 NFC bytes of `"şifre-ñ"` directly — Myne deliberately does NOT normalize passwords. The wrap-key bytes for this case differ from `recovery-nfkd-unicode` even with the same byte length and salt structure, because the salt prefix differs.

## Regeneration

If the reference implementation changes (must not happen byte-stably without a major-version event), regenerate this vector set via:

```bash
cargo test -p protocol --lib emit_argon2id_wrap_vectors_for_paste -- --ignored
# Allow ~25 seconds; 7 cases × ~3s per Argon2id derivation at canonical params.
cp /tmp/myne_argon2id_vectors.json vectors/argon2id/v0.1/cases.json
```

The regenerator test lives in `protocol/src/kdf.rs::tests::emit_argon2id_wrap_vectors_for_paste`. `#[ignore]`-gated — regeneration is intentional, not a side effect of `cargo test`.

## What this does NOT cover

- **Argon2id parameter range edge cases.** Out-of-range `m_kib` / `t` / `p` values, unsupported `argon2_version` bytes (anything other than `0x10` / `0x13`) — covered by in-crate unit tests in `protocol/src/kdf.rs` (`derive_with_invalid_*` family). Not cross-impl conformance vectors because the failure semantics are language-agnostic and not byte-level.
- **Phrase NFKD normalization itself.** The `phrase_normalize` vector category pins the input normalization step independently. This category exercises Argon2id on the post-normalization bytes; combined, they verify the full phrase → wrap-key pipeline at byte level.
- **Mobile-profile parameter set.** A future v0.1.x mobile profile may use lower memory parameters (e.g., m=64 MiB) for resource-constrained devices. Vectors at those parameters would land in their own sub-directory with the relaxed schema `const`.
- **The `derive_recovery_wrap_key` (structured `RecoveryPhrase` input) entry point.** That function just delegates to `_from_normalized` after calling `phrase.join_normalized()`. Both functions are byte-equivalent given byte-equivalent normalized input; this category exercises the `_from_normalized` entry point directly to avoid coupling the wrap-key vectors to the `RecoveryPhrase` struct constructor.
