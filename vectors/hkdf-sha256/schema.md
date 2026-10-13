# `hkdf-sha256` vector schema

Pins the byte mapping of `protocol::derive_subkey` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)) — the canonical HKDF-SHA-256 helper through which every v0.1 subkey is derived from `root_key`. Reference implementation: `protocol/src/subkey.rs::derive_subkey`. Spec: `docs/spec/protocol.md` § 3.2 (HKDF construction) + § 3.4 (role registry).

## Pipeline (as fixed by spec § 3.2)

For each `(root_key, role)` input pair:

1. **Construct `info`** as the byte concatenation `"myne.v0.1 " || role.role_str()` — no separator, no length prefix. The 10-byte prefix `"myne.v0.1 "` (note the trailing space, mandated by spec § 1.2 + § 3.2.2) is part of the domain-separation contract; any drift silently invalidates every existing vault.
2. **HKDF-Extract**: `PRK = HMAC-SHA256(salt = zero_32, ikm = root_key)`. The salt is 32 zero bytes — deterministic on `root_key`.
3. **HKDF-Expand**: `subkey = HKDF-Expand(PRK, info, L = 32)`. Output is exactly 32 bytes.

The output is wrapped in `Zeroizing<[u8; 32]>` in the Rust reference so the bytes are best-effort wiped on drop. Implementations in other languages SHOULD provide an equivalent zeroize-on-drop wrapper.

## Role registry (spec § 3.4)

v0.1 registers six roles:

| `input_role` string       | Rust variant                          | Where it's used |
|---------------------------|---------------------------------------|-----------------|
| `itemsKey-wrap`           | `SubkeyRole::ItemsKeyWrap`            | Wraps the singleton `itemsKey` (spec § 3.3, § 9.3 offset 281) |
| `prefs-key`               | `SubkeyRole::PrefsKey`                | KEK for the `vault.prefs` blob (spec § 3.4, § 9.5)            |
| `search-index-key`        | `SubkeyRole::SearchIndexKey`          | KEK for the `vault.search` blob (MYND-29, § 9.6) — distinct from the AAD role `search-index-blob` |
| `attachments-catalog-key` | `SubkeyRole::AttachmentsCatalogKey`   | KEK for the `vault.attachments` catalog (MYND-164, § 9.10)    |
| `attachments-id-hmac`     | `SubkeyRole::AttachmentsIdHmac`       | Keys the HMAC-SHA-256 deriving content-addressed attachment-ids (§ 9.9) — NOT a blob KEK; § 9.10.1 domain-separation pair with the catalog KEK |
| `session-state-key`       | `SubkeyRole::SessionStateKey`         | KEK for the `vault.workspace` blob (MYND-254, § 9.7) — distinct from the AAD role `workspace-blob` |

Future v0.1.x revisions register additional roles by appending to the type-safe `SubkeyRole` enum AND the schema's `input_role` enum in lockstep — additive only; existing variant strings are immutable.

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level shape) and [`schema/v0.1/hkdf-sha256.json`](../../schema/v0.1/hkdf-sha256.json) (per-case shape; single `case_kind`, no discriminator). Run the repository's documented `ajv-cli validate` command (see the root `README.md`) to verify any `cases.json` against the canonical schema. The sub-schema is authoritative — this document covers only the domain-specific semantics.

Each case pins one `(input_root_key_hex, input_role)` → `expected_subkey_hex` mapping. The optional `informational_info_utf8_hex` field, when present, carries the canonical info bytes (`"myne.v0.1 " || role_str`, hex-encoded) so independent implementers verifying their port can confirm the info-parameter assembly without re-parsing the spec — verified implicitly via the `expected_subkey_hex` assertion.

## How to consume

A port runs each case as: decode `input_root_key_hex` (32 bytes), map `input_role` to the port's `SubkeyRole` equivalent (rejecting any unknown role string), invoke the port's `derive_subkey` equivalent, hex-encode the 32-byte output, compare byte-equal to `expected_subkey_hex`. Any non-equal result is a stop-the-world bug (per repo `README.md` "Cross-implementation discipline").

## Invariants the vectors pin

For the 48 cases in `cases.json` (the runner iterates `SubkeyRole::ALL`, so this count follows the registry rather than pinning it):

- **HKDF prefix is exactly `"myne.v0.1 "`.** Trailing space is load-bearing. Drift here surfaces as a mismatch across every vector.
- **Role-string registry is exact.** The six registered strings only — case-sensitive, no whitespace, no separator. The `informational_info_utf8_hex` field documents the exact info-parameter assemblies (23 B `itemsKey-wrap`, 19 B `prefs-key`, 26 B `search-index-key`, 33 B `attachments-catalog-key`, 29 B `attachments-id-hmac`, 27 B `session-state-key`, each including the 10-byte prefix).
- **Cross-role independence.** The all-zero root and the alpha pseudorandom root each appear with multiple roles; the outputs MUST differ byte-for-byte (includes the § 9.10.1 attachments domain-separation pair). Enforced implicitly via the byte-equality of each independent vector.
- **Determinism.** `derive_subkey` is pure; re-running against the same `(root_key, role)` yields bit-identical output. Enforced implicitly via `cargo test` determinism.
- **Output length is always 32 bytes.** HKDF-Expand produces 32 bytes regardless of role or root.

## Regeneration

If the reference implementation changes (must not happen byte-stably without a major-version event), regenerate this vector set via:

```bash
cargo test -p protocol --lib emit_subkey_vectors_for_paste -- --ignored
cp /tmp/myne_hkdf_sha256_vectors.json vectors/hkdf-sha256/v0.1/cases.json
```

The regenerator test lives in `protocol/src/subkey.rs::tests::emit_subkey_vectors_for_paste`. It is `#[ignore]`-gated so it never runs in default `cargo test` invocations. Unlike the paste-format generators for `vault-prefs` / `vault_meta_rewrite`, this one emits ready-to-copy JSON for the whole file, because assembling a 48-case matrix by hand is how the appends below went missing in the first place.

**This is a WHOLE-FILE generator, and that is the hazard.** The command above replaces `cases.json` in one act. It is NOT append-safe the way `vault-prefs` is, and the file has in fact been extended by appending one feature's roles at a time rather than by re-emitting all of it:

- the **all-zero** root pairs, covering the roles that predate the alpha root;
- the **alpha** pseudorandom root, whose pairs cover the sync-family and Phase-2 roles;
- a third root, exercised by `vault-head-mac` and `server-auth`;
- the **`chats-*` appends** (`chats-catalog-key`, `chats-catalog-sync-id`, `chats-key`; myne-md/myne#675) and the **`agent-*` appends** (`agent-memory-catalog-key`, `agent-memory-catalog-sync-id`, `agent-memory-key`, `agent-personality-key`, `agent-personality-sync-id`; myne-md/myne#697), all eight derived from one input: `[0x42; 32]`, a literal byte pattern rather than a `pseudorandom_root(...)` seed. **Their `vector_id` slugs carry no root prefix**, unlike every earlier case, which is what makes them identifiable as the appended set.

The first three landed with the emitter; the two appends did not. For two releases the documented regeneration command produced a **40-case** document against a **48-case** file, and pasting its output would have deleted eight live roles' vectors. That was invisible from either side alone: the emitter type-checked and ran clean, and the companion's `ajv` sweep validates a 40-case file as happily as a 48-case one. The only thing that made it loud is the runner's coverage pin over `SubkeyRole::ALL` in the main repo's `protocol/tests/vectors.rs`, which names the missing role instead of reporting a pass.

**So the obligation is on the contributor, not on a hand-splice.** A landing that registers a new `SubkeyRole` must extend this emitter's array in the same change; the count in `let cases: [Case<'_>; N]` is a compile-time floor, but nothing mechanical ties that number to the registry. Check it by running the emitter and diffing **by id, not by length**: paste its stdout over the file, then confirm the case count and the id set are unchanged from the committed file. If the diff drops an id, the emitter lost a case and the fixture must not be overwritten.

## What this does NOT cover

- **Tampered info-string vectors.** The Rust reference's type-safe `SubkeyRole` enum makes a tampered info parameter inexpressible at compile time — there's no construction site through which a port could pass a wrong info string and have it survive review. In-crate unit tests at `protocol/src/subkey.rs:174,188` exercise the underlying `derive_with_context` (a `pub(crate)` helper) with intentionally tampered info bytes to confirm the contract; those tests stay in-crate because they're not portable contract assertions.
- **The full subkey usage flow.** This directory pins only the derivation. Vectors that exercise the derived subkeys' downstream usage (sealing `itemsKey`, encrypting `vault.prefs` plaintext) live in `vectors/vault-prefs/` + `vectors/ctx-aead/` and the planned `vectors/aad-construction/`.
- **Argon2id-derived roots.** The Phase 1 v0.1 root-key derivation (password → root_key via Argon2id) is pinned in the planned `vectors/argon2id/` directory; the `input_root_key_hex` values here are constants chosen to exercise the HKDF construction independently of the upstream KDF.
