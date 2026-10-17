# `damm32-table` vector schema

Pins the **frozen Damm-32 order-32 quasigroup table** that backs the account-number check character (`account-number.md` §4.2), as a language-agnostic ground truth. An independent port derives its own order-32 quasigroup over GF(2⁵) and diffs it, cell-for-cell, against `expected_table_hex` — without trusting the reference Rust. This is the **only** account-identity artifact that is independent of a port's own field-arithmetic implementation.

**Reference implementation:** `protocol/src/account_number.rs` — `damm32_table()` (the frozen `DAMM32_TABLE` const) and `damm32_step` (the equivalent runtime construction).
**Spec:** `docs/spec/account-number.md` §4 (§4.1 construction, §4.2 normative frozen table) + `design notes.md` 2026-04-28 "Damm-32 polynomial form" (the construction anchor) and its table-freeze follow-on.

## Construction (informative — the table is the contract)

The quasigroup is built over the finite field GF(2⁵) ≅ GF(2)[x]/(x⁵ + x² + 1):

```
x ∗ y = α · (x ⊕ y)        α = the field element 2 (the polynomial x)
step(interim, symbol):     # one Damm-32 step
  t := (interim XOR symbol) << 1
  if t >= 32: t := t XOR 37     # 37 = 0b100101 = x⁵ + x² + 1
  return t
table[interim][symbol] := step(interim, symbol)
```

The polynomial `x⁵ + x² + 1` is the canonical low-weight (and primitive) irreducible of degree 5 (Seroussi, HP Labs HPL-98-135). The operation is commutative, so the quasigroup is **weak** totally anti-symmetric with zero diagonal — the precise (and sufficient) requirement for the Damm check equation. See `account-number.md` §4.1.

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/damm32-table.json`](../../schema/v0.1/damm32-table.json) (per-case shape, 2-way `case_kind` discriminator: `table`, `checksum`). Run the repository's documented `ajv-cli validate` command to verify `cases.json` against the canonical schema.

- **`table`** — the full table. `expected_table_hex`: 1024 bytes (2048 hex chars), **row-major** — byte at offset `32*interim + symbol` is `table[interim][symbol]`, each a Crockford index 0..31. `informational_polynomial` / `informational_alphabet` / `informational_provenance` document the derivation (the provenance note records that the table matches the published `srobo/damm32` (MIT) reference — provenance/citability, not an independent-correctness claim).
- **`checksum`** — a running-checksum example. `input_entropy_indices`: a left-to-right sequence of 0..31 indices folded from interim 0 (`interim := table[interim][index]`). `expected_check_index`: the final interim (the Damm-32 check index). Pins direction (left fold) and orientation (row = interim, column = symbol).

## How to consume

- **`table`**: build your order-32 quasigroup, serialize it row-major to bytes, hex-encode, assert equality with `expected_table_hex`. Any divergence is a stop-the-world bug — your field arithmetic, polynomial, or index order has drifted.
- **`checksum`**: fold `input_entropy_indices` from interim 0 through your table and assert the result equals `expected_check_index`.

## Invariants the vectors pin

- **The 1024-byte table is byte-identical across implementations.** Orientation (`table[interim][symbol]`, row-major), polynomial (37), and Crockford index order are all load-bearing — a transposed table or wrong polynomial produces a different table and a different account-identity space.
- **Checksum direction is a left fold from interim 0.** `checksum-single-step` pins `table[0][1] = 2`; `checksum-baseline-19` pins indices `0..18 → 26` (the account-number baseline whose check char is `T`); `checksum-all-high-19` is a far-from-baseline stress case.
- **Changing this table is a major-version event.** It would reject already-distributed account numbers and break cross-impl validity agreement (root `AGENTS.md` load-bearing registry).

## Relationship to `account-number`

The [`account-number`](../account-number/) category pins `parse_account_number` behavior (full 20-character strings: liberal-parse normalization + Damm-32 substitution/transposition catches). This category pins the underlying **table + raw index checksum** — complementary, lower-level ground truth. A port that passes `damm32-table` but fails `account-number` has a parsing/normalization bug; the reverse points at the table itself.

## Regeneration

```bash
cargo test -p protocol --lib emit_damm32_table_vectors_for_paste -- --ignored
cp /tmp/myne_damm32_table_vectors.json vectors/damm32-table/v0.1/cases.json
```

The regenerator (`protocol/src/account_number.rs::tests::emit_damm32_table_vectors_for_paste`, `#[ignore]`-gated) cross-checks the table against an independent transcription of the published `srobo/damm32` (MIT) step at mint time, so the `informational_provenance` claim is true by construction. The companion `emit_damm32_table_for_paste` regenerates the in-crate const and the spec §4.2 markdown block.

## What this does NOT cover

- **Full account-number parsing / normalization** — that is the `account-number` category.
- **The `generate_account_number` CSPRNG path** — non-deterministic, not pinnable.
- **Cross-major tables** — a later major version may redefine the polynomial or entropy length; those would land under `vectors/damm32-table/<version>/` beside `v0.1/`. (The next major is `v0.2`; the retired `v005` codename is recorded in `design notes.md` 2026-08-01.)
