# `account-number` vector schema

Pins the byte mapping of `protocol::parse_account_number` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)) — the liberal-parse normalizer that converts user-typed account-number strings into a canonical 20-character Crockford-base32 form + 16-byte hash. Together with `generate_account_number` (not exercised in this category because it pulls from CSPRNG) these define the v0.1 account-identity contract.

**Reference implementation:** `protocol/src/account_number.rs::parse_account_number`.
**Spec:** `docs/spec/protocol.md` (high-level binding into Argon2id salts + `vault.meta` offset 6) + `docs/spec/account-number.md` § 3 (canonical form definition) + § 5 (liberal-parse normalization pipeline) + `design notes` 2026-04-28 entry "Account-number Damm-32 polynomial form freeze".

## Pipeline (as fixed by `account-number.md` § 5)

For each input candidate string:

1. **NFKD normalization** of the entire input.
2. **Lowercase + strip whitespace** (case-insensitive; surrounding and interior whitespace removed).
3. **Strip optional `MY-` prefix** (case-insensitive).
4. **Strip hyphens** (`-` separators between groups are optional).
5. **Validate length** = `CANONICAL_LEN` (20 chars) after the above normalizations.
6. **Validate each char** against the Crockford-base32 alphabet `0123456789ABCDEFGHJKMNPQRSTVWXYZ` STRICTLY — **no `I → 1` / `L → 1` / `O → 0` / `U → V` lookalike substitutions per spec § 5 step 7**. The Rust parser uses the alphabet exactly as documented; `I`, `L`, `O`, `U` are all rejected.
7. **Validate Damm-32 check digit** — entropy[0..18] running through the canonical Damm-32 polynomial must yield the value at entropy[19]. Polynomial form is frozen per `design notes` 2026-04-28.

Any rule failure surfaces as a single non-distinguishing `ProtocolError::AccountNumberInvalid` — ports MUST NOT distinguish which step fired beyond a single user-facing 'invalid account number' message (spec § 5 step 8 closed-error discipline).

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/account-number.json`](../../schema/v0.1/account-number.json) (per-case shape with a 2-way `case_kind` discriminator: `parse_ok`, `parse_fail`). Run the repository's documented `ajv-cli validate` command (see the root `README.md`) to verify any `cases.json` against the canonical schema. The sub-schema is authoritative — this document covers only the domain-specific semantics.

Two case kinds carry different field shapes:

- **`parse_ok`** — pins the liberal-parse output. Inputs: `input_candidate_utf8_hex` (the candidate string, UTF-8 bytes hex-encoded). Outputs: `expected_canonical_utf8_hex` (20-byte canonical form as uppercase Crockford ASCII), `expected_display_utf8_hex` (27-byte display form with `MY-` prefix + group hyphens), `expected_hash_hex` (16-byte `SHA-256(canonical_ascii)[:16]`).
- **`parse_fail`** — pins the strict-reject surface. Input: `input_candidate_utf8_hex`. Output: `expected_kind: "parse_fail"`. Implementations MUST return their port's equivalent of `Err(ProtocolError::AccountNumberInvalid)` — single non-distinguishing variant per spec § 5 step 8. The optional `informational_failure_reason` field documents which rule fired for human-readable audit; runners MUST NOT depend on it.

## How to consume

A port runs each case based on `case_kind`:

- **`parse_ok`**: decode `input_candidate_utf8_hex` to bytes, parse as UTF-8 string, invoke the port's `parse_account_number` equivalent, assert the returned `(canonical, display, hash)` triple matches the corresponding `expected_*_hex` fields byte-for-byte. Any divergence is a stop-the-world bug.
- **`parse_fail`**: decode inputs, invoke the parser, assert it returns the non-distinguishing AccountNumberInvalid-equivalent variant. Ports MUST NOT distinguish which step rejected.

## Invariants the vectors pin

For the 12 cases in `cases.json`:

- **Six liberal-parse permutations all map to the same canonical / display / hash triple.** Canonical: `0123456789ABCDEFGHJT`. Display: `MY-0123-4567-89AB-CDEF-GHJT`. Hash: SHA-256 truncation; deterministic from canonical. Implementations whose liberal-parse normalizes any of these 6 inputs differently are broken.
- **Strict Crockford alphabet — no lookalike substitution.** Vectors `non-crockford-char-O` and `non-crockford-char-I` exercise the spec § 5 step 7 strict policy: chars not in `0123456789ABCDEFGHJKMNPQRSTVWXYZ` are rejected, NOT silently mapped to similar-looking valid chars. This is intentional — a user who types `O123...` should be told the input is wrong, not silently coerced.
- **Length is exactly 20 after normalization.** Vectors `wrong-length-too-short` (18) and `wrong-length-too-long` (21) prove the length check fires.
- **Damm-32 catches all single-character substitutions** — vector `damm32-substitution-catch`. This is a property of the Damm algorithm, not the implementation; if your port's Damm-32 doesn't catch single substitutions, the polynomial form has drifted from the 2026-04-28 freeze.
- **Damm-32 catches all adjacent-pair transpositions** — vector `damm32-transposition-catch`. Same property; same drift signal.

## Regeneration

If the reference implementation changes (must not happen byte-stably without a major-version event), regenerate this vector set via:

```bash
cargo test -p protocol --lib emit_account_number_vectors_for_paste -- --ignored
cp /tmp/myne_account_number_vectors.json vectors/account-number/v0.1/cases.json
```

The regenerator test lives in `protocol/src/account_number.rs::tests::emit_account_number_vectors_for_paste`. It is `#[ignore]`-gated so it never runs in default `cargo test` invocations. Emits ready-to-copy JSON (matches the recently-introduced `subkey.rs` / `vault_meta.rs` pattern).

## What this does NOT cover

- **`generate_account_number` CSPRNG path.** That function uses `getrandom` and produces a fresh random account number each invocation — non-deterministic and not pinnable as a byte-level vector. The CSPRNG-availability path is exercised by in-crate unit tests; the byte-level interop contract is fully captured by `parse_account_number` cases.
- **NFKD + full-width char + no-break-space normalization edge cases.** Exercised by the in-crate unit tests `parse_accepts_fullwidth_*` and `parse_accepts_no_break_space_separator`. Future expansion may promote selected NFKD cases here if cross-impl drift is observed (e.g., a port that doesn't NFKD-normalize would fail those in-crate tests, but cross-impl conformance gains from explicit byte-level vectors).
- **AccountNumber → Argon2id salt binding.** That's exercised by the future `argon2id` vector category (MYND-69), which uses `account_number_hash` as a binding input to the canonical Argon2id wrap derivations.
- **Cross-major vectors.** A later major version may redefine the entropy length or the polynomial; those would land under `vectors/account-number/<version>/` beside `v0.1/`, validated by that version's own root schema. (The next major is `v0.2`; the `v005` codename this line once used is retired — `design notes` 2026-08-01.)
