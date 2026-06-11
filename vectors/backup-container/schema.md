# `backup-container` vector schema

Pins the byte mapping of the `MYB1` `.myne-backup` whole-vault backup container (codec landed MYND-217, vectored MYND-254): the deterministic write side, the credential-free verifier, and the § 9.11.3 closed-allowlist path classifier.

**Reference implementation:** `src-tauri/src/backup_container.rs::{write_container, read_and_verify_container, validate_member_path, classify_member_path}` (main [`myne` repo](https://github.com/myne-md/myne), package `myne-desktop`). This is the **first vector family whose reference implementation lives outside the `protocol` crate**: the codec is std/fs/streaming-bound by design (file-path API, cooperative-cancel hooks) and cannot move into the `no_std` protocol crate. The generator and runner therefore live in `backup_container.rs`'s test module rather than `protocol/` — see Regeneration below.
**Spec:** `docs/spec/protocol.md` § 9.11 (container layout § 9.11.1, member-table record + hash formula § 9.11.2, closed member allowlist § 9.11.3, integrity semantics § 9.11.4, validation order + ceilings § 9.11.6).

## Scope: a credential-free, UNKEYED integrity layer

Two boundaries a porter must keep straight:

- **No cryptography of its own.** Members are byte-for-byte ciphertext copies of vault artifacts at their exact vault-relative paths; the container never parses member content and never touches key material. Per-member SHA-256 hashes are **unkeyed** — they detect *accidental corruption and truncation only*, never tampering (an adversary recomputes them trivially). Tamper defense is the members' own AEAD at first unlock (§ 9.11.4). Member paths are themselves load-bearing: `.myn` filenames are AAD-bound note UUIDs and attachment filenames are content-addressed ids, so a renamed/transformed member fails AEAD verification downstream.
- **Container layer only (stages 1–4).** These vectors pin the § 9.11.6 credential-free validation stages: header, ceilings + length equations, member table, per-member hashes. Stage 5 — the embedded `vault.meta` REQUIRED-presence check and its exact-345-byte structural parse — belongs to the restore engine and is **not** covered here. Consequently the `vault.meta` family cap pinned here is `<= 345` (an allocation bound), not exact-345.

## Container layout (as fixed by the reference implementation)

```
Offset  Size  Field              Notes
------  ----  -----------------  -----------------------------------
0       4     magic              ASCII "MYB1"
4       1     protocol_version   0x01
5       1     reserved           0x00 (MUST)
6       4     member_count       N, big-endian u32
10      8     members_total_len  Σ member_len, big-endian u64
18      4     member_table_len   T, big-endian u32
22      T     member table       N records, sorted by raw path bytes
22+T    ...   member data        concatenated, table order, no padding
```

Record: `path_len (u16 BE) || path || member_len (u64 BE) || sha256 (32 B)`. The SHA-256 covers `path_len_be16 || path || member bytes` — **path-inclusive**, so a corrupted path byte that still parses as allowlist-valid is as detectable as data corruption.

**Canonical encoding:** records MUST be sorted lexicographically by raw path bytes, duplicates rejected, data laid out in table order — the same member set always encodes to identical container bytes regardless of producer input order. The `encode` cases present their `input_members` deliberately **unsorted** to pin that the writer owns the sort.

**Consistency equations (§ 9.11.6):** `file_length == 22 + member_table_len + members_total_len`; `member_table_len == Σ (42 + path_len)`; `members_total_len == Σ member_len`. Reference ceilings: `member_count ≤ 2^20`, `member_table_len ≤ 16 MiB`, `path_len 1..=1024`.

**Validation order is load-bearing:** header checks (magic / protocol_version / reserved), then the two ceilings, **then** the file-length equation, then table parse (path rules → allowlist → order/dups → family caps), then hashes. The two ceiling cases are 22-byte header-only files: a port that checks the file-length equation before the ceilings would mislabel them `lengths_inconsistent`.

## Member allowlist (§ 9.11.3, closed)

| Family (`expected_family`) | Path grammar | Cap (allocation bound) |
|---|---|---|
| `vault-meta` | `vault.meta` (REQUIRED — enforced at stage 5, not here) | 345 B |
| `vault-prefs` | `vault.prefs` | 64 KiB + 34 + 32 |
| `note` | `notes/<uuid>.myn` | 17 MiB |
| `snapshot` | `snapshots/<note-uuid>__<snap-uuid>.myn` | 17 MiB |
| `attachments-catalog` | `vault.attachments` | 17 MiB |
| `attachment-blob` | `attachments/<64-lowercase-hex>` | 65 MiB |

`<uuid>` is the strict disk form: 36 chars, hyphens at 8/13/18/23, lowercase hex elsewhere — uuid-library leniency (uppercase, braces) MUST NOT leak into the grammar. Reserved OUT: `vault.search` (rebuildable cache), `vault.workspace` (rebuildable session state), `*.tmp` residue, `registry.json` (vault-external). Writers MUST NOT include out-of-allowlist paths; readers MUST reject them (not skip).

## File layout

```
vectors/backup-container/
├── schema.md              ← this file
└── v0.1/
    └── cases.json         ← array of 32 KAT cases (encode + verify_ok + verify_fail + classify_path)
```

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/backup-container.json`](../../schema/v0.1/backup-container.json) (per-case shape with a 4-way `case_kind` discriminator). The sub-schema is authoritative — this document covers only the domain-specific semantics.

- **`encode`** — pins the deterministic member-set → container-bytes mapping. Inputs: `input_members` (array of `{path, bytes_hex}` in deliberately arbitrary order). Output: `expected_container_hex`.
- **`verify_ok`** — pins the verifier's parse. Input: `input_container_hex`. Output: `expected_member_records` (array of `{path, len, sha256_hex}` in table order; the `sha256_hex` values independently pin the § 9.11.2 hash formula).
- **`verify_fail`** — pins a credential-free reject. Input: `input_container_hex`. Output: `expected_error`, one of: `header_too_short`, `magic_mismatch`, `protocol_version_mismatch`, `reserved_byte_nonzero`, `member_count_overflow`, `member_table_overflow`, `path_invalid`, `path_not_allowlisted`, `duplicate_path`, `path_order_violation`, `member_len_overflow`, `lengths_inconsistent`, `hash_mismatch`. The `MYB5` reserved v005 magic appears as a second `magic_mismatch` case — a v0.1 consumer MUST reject it (the desktop caller re-checks the bytes to phrase a migration prompt).
- **`classify_path`** — pins the § 9.11.3 grammar from both directions. Input: `input_member_path` (plain string). Output: `expected_family` (a family name from the table above, or `rejected`).

## How to consume

A port runs each case based on `case_kind`:

- **`encode`**: feed the member set (in the order given — do NOT pre-sort) to the port's container writer, compare output byte-equal to `expected_container_hex`.
- **`verify_ok`**: run the port's credential-free verifier over the container bytes, compare the resulting `(path, len, sha256)` triples in order.
- **`verify_fail`**: run the verifier, assert it surfaces the *specific* reject named by `expected_error` — including the stage-order cases.
- **`classify_path`**: run the port's path validation + allowlist classification, compare the family (or rejection).

Any mismatch is a stop-the-world bug (per repo `README.md` "Cross-implementation discipline").

## Invariants the vectors pin

- **Header byte layout + the three big-endian length fields** (§ 9.11.1), via both `encode` and the tampered `verify_fail` derivatives.
- **Canonical encoding**: unsorted input → sorted table → identical bytes; data order = table order (`encode-full-allowlist-canonical-order`).
- **The path-inclusive hash formula** `SHA-256(path_len_be16 || path || member_bytes)` — independently checkable from `verify_ok`'s `sha256_hex` values.
- **The § 9.11.6 validation order**: ceilings before the file-length equation (`verify-fail-member-count-overflow` / `verify-fail-member-table-overflow` are header-only files).
- **The MYB5 v005 reservation**: refused by a v0.1 consumer, collapsing to `magic_mismatch`.
- **The closed allowlist from both directions**: every family accepted at least once; the reserved-out singletons (`vault.search`, `vault.workspace`) and the grammar edges (uppercase hex, 63-char id, uppercase UUID, missing `__` pair, `.tmp` suffix) rejected.
- **Family caps from the read direction**: a 346-byte `vault.meta` member rejects as `member_len_overflow` at table parse, with all length equations holding.

## Regeneration

If the reference implementation changes (must not happen byte-stably without a major-version event), regenerate this vector via:

```bash
cargo test -p myne-desktop --lib emit_backup_container_vectors_for_paste -- --ignored
cat /tmp/myne_backup_container_vectors.json
# copy the full JSON verbatim into cases.json
```

Note the package: `-p myne-desktop`, NOT `-p protocol`. The regenerator test lives in `src-tauri/src/backup_container.rs::tests::emit_backup_container_vectors_for_paste` and self-asserts every expectation at mint (every `verify_fail` fixture is run through `read_and_verify_container`, every `classify_path` case through the classifier). It is `#[ignore]`-gated so it never runs in default `cargo test` invocations. The matching runner (`backup_container_vectors`, same test module) follows the `protocol/tests/vectors.rs` skip-if-missing discipline.

## What this does NOT cover

- **Runtime-only error modes.** `BackupContainerError::Cancelled` (cooperative cancel) and `Io(...)` (filesystem failure) are not byte-derivable and cannot be vectored.
- **Stage 5.** The `vault.meta` REQUIRED-presence check and its exact-345-byte § 9.3 structural parse are the restore engine's job (MYND-218) — verified by `src-tauri` integration tests, not cross-implementation vectors. (The 345-byte fixture here is an opaque `0xAB` fill, *not* a parseable vault.meta — deliberately, to pin the container layer's content-agnosticism.)
- **The phased two-pass writer** (`write_container_phase_one` + `PhasedWriter::finish`, the SYNC-6 bounded-lock-hold refinement). Its output is byte-identical to the inline writer by construction; the `phased_write_equals_inline_write_byte_for_byte` unit test owns that equivalence.
- **Export/restore engine behavior**: writer-quiescence locking, read-back-verify-before-rename (SYNC-4), salvage mode's skipped-member reporting (SYNC-5d), staging-directory atomicity — all `src-tauri` integration-test territory.
- **Point-in-time member-set consistency** (§ 9.11.5) — a producer-side obligation invisible in the wire format.
