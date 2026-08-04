<p align="center"><img src=".github/banner.png" alt="Myne test-vectors" width="100%"></p>

# Myne test-vectors

> One byte-exact contract. Every implementation agrees.

Language-agnostic JSON test vectors that pin the [Myne](https://github.com/myne-md/myne) v0.1 cryptographic protocol at the byte level. Any independent implementation — Rust, Go, Swift, Kotlin, JavaScript, or anything else — must produce bit-identical outputs from these inputs. A reviewer verifying a port should not have to read the reference Rust implementation; running their own language's test runner against this repository is enough.

This is a companion repo of the [Myne multi-repo project](https://github.com/myne-md). The normative protocol lives in the main repo's specs — start with [`docs/spec/protocol.md`](https://github.com/myne-md/myne/blob/main/docs/spec/protocol.md), [`docs/spec/account-number.md`](https://github.com/myne-md/myne/blob/main/docs/spec/account-number.md), and the cross-implementation magic-byte and identifier tables. These vectors are the executable projection of that spec: every magic prefix, every AEAD AAD framing, every subkey role, and every coercion rule encoded as inputs and expected outputs.

## Status

**Pre-alpha**, tracking the v0.1 protocol. This repo holds data and schemas only — there is no build step and no code to compile here.

What exists today:

- **50 vector categories**, each under `vectors/<category>/` with a domain-semantic `schema.md` and a JSON-Schema-validated `v0.1/cases.json`.
- A **canonical machine-readable contract**: the root schema `schema/v0.1.json` dispatches per-category sub-schemas under `schema/v0.1/` (one file per category) via `oneOf` keyed on the `schema` field.
- **Git hooks** (`lefthook.yml`) that guard JSON well-formedness and text hygiene on commit.

What is deferred:

- **CI integration.** Schema validation runs as a developer-side step today; there is no pipeline in this repo yet.
- **v005 vectors.** The next major epoch is reserved but unwritten — v0.1 and v005 are forward-incompatible by design.

Every v0.1 blob family in the [`protocol.md` §10.3 magic-allocation table](https://github.com/myne-md/myne/blob/main/docs/spec/protocol.md) now has a category, every `SubkeyRole` variant has an `hkdf-sha256` case, and every vault-scoped `aad_*` builder has an `aad-construction` decomposition. Those three registries are enforced from the runner side — `SubkeyRole::ALL` and the builder lists are iterated, not restated — so a new role or family fails the suite instead of quietly shipping unvectored.

## How it works

Myne is end-to-end encrypted and local-first: a 20-character account number identifies you (no email), BIP-39 24-word recovery, Argon2id + XChaCha20-Poly1305. The same protocol runs whether sync is hosted or self-hosted — they share one trust boundary (TB5 in the threat model), so a vector that passes for one passes for both. These vectors are how independent clients prove they agree on the wire and on disk, down to the byte.

Each category pins one surface of the protocol. They fall into a few shapes:

- **On-disk blob codecs** — encrypt, decrypt-success, non-distinguishing AEAD-failure, and distinguishable structural parse-reject cases for each magic-prefixed format (`MY01` notes, `MYV1` vault metadata, `MYP1` prefs, `MYS1` search, `MYW1` workspace, `MYI1` indices, `MYA1`/`MYC1` attachments, `MYD1` dictionary, `MYQ1` quick-unlock, `MYE1` attachment-text, `MYB1` backup container, `MYR1` AI embeddings, `MYG1` graph layout, `MYK1` pending capture, `MYF1` share import, `MYT1` custom themes). `MYH1` (`vault.head-state`) is the one family in this group that is **not** AEAD-encrypted — integrity is the vault head's own MAC — so it pins a `mac_fail` case kind instead of an AEAD-failure one.
- **Primitive derivations** — HKDF-SHA-256 subkey roles, Argon2id wrap-key derivations at canonical Phase 1 parameters, and the XChaCha20-Poly1305 + CTX-commitment AEAD with every AAD builder's canonical byte assembly.
- **Identity** — Crockford-base32 account-number parsing plus the frozen Damm-32 quasigroup table that decides which account numbers are valid.
- **Deterministic conformance** — non-crypto, cross-client behavior such as phrase normalization, the password-length floor, the BIP-39 recovery composition, the note-template token engine, frontmatter property coercion, the `%%…%%` editor-comment recognizer, the clip serializer's canonical emit, the dialect parse-recognition categories (`footnote` / `highlight` / `wikilink` / `definition-list` / `inline-tag` / `math`), the task-field extension across all three read dialects and the one emitted form (`task-inline-format`) the closed recurrence vocabulary and its DST-immune civil-date arithmetic (`task-recurrence`), plus the closed task-query roster and the clauses it refuses (`task-query-grammar`), the frontmatter field patchers (`frontmatter-surgery`), the saved-views + color-group + pairing-token array decoders (`saved-view` / `color-group` / `paired-extension`), and the rename-journal crash-recovery state machine (`rename-journal`).

Vectors are not hand-typed where a generator exists: most are emitted by `#[ignore]`-gated generator tests in the main repo (`protocol/`, or `src-tauri/` for the std/fs-bound `backup-container` + `rename-journal`), so regeneration is always a deliberate act. The TS-runner categories are emitted by env-gated vitest generators (`d1-clip-serializer`, `footnote`/`highlight`/`wikilink`/`definition-list`/`inline-tag`/`math` via `GEN_DIALECT_VECTORS=1`, `frontmatter-surgery` via `GEN_FM_SURGERY=1`, `task-inline-format`/`task-recurrence` via `GEN_TASK_VECTORS=1`) or hand-curated against the spec (`phrase_normalize`, `property-coercion`, `markdown-comment`); all are asserted by TypeScript runners.

## Structure

```
myne-test-vectors/
├── schema/
│   ├── v0.1.json              ← root schema (top-level shape + oneOf dispatch on `schema`)
│   └── v0.1/
│       └── <category>.json    ← one per-category case schema (50 files)
├── vectors/
│   └── <category>/
│       ├── schema.md          ← domain-semantic description of the category
│       └── v0.1/cases.json    ← the vectors themselves
├── lefthook.yml               ← commit-time JSON + text-hygiene hooks
└── LICENSE                    ← CC-BY-4.0
```

Adding a new category requires landing its sub-schema under `schema/v0.1/` and extending both the `schema` enum and the `oneOf` branches in the root schema in lockstep, then adding the `vectors/<category>/` files and a runner in the main repo.

### Field conventions

Every `cases.json` follows the same naming convention so a port can consume any category mechanically:

- Identity: `vector_id` (string, `<category>/<version>/<descriptive-slug>` — immutable once shipped).
- Discriminator: `case_kind` (present only when a category has more than one case shape).
- Inputs: `input_<semantic>_<encoding>` (e.g. `input_root_key_hex`, `input_plaintext_utf8_hex`).
- Expected outputs: `expected_<semantic>_<encoding>` (e.g. `expected_subkey_hex`).
- Failure-mode expectation: `expected_kind` (e.g. `aead_fail`).
- Informational / intermediate: `informational_<semantic>[_<encoding>]`.
- `protocol_version: "v0.1"` is pinned once at the top of each `cases.json` (alongside `schema`, `description`, and `spec_section`) so future v005 vectors can coexist without ambiguity.

Hex-encoded values are lowercase with no `0x` prefix.

## Validating vectors

There is no package to install. Validate every landed `cases.json` against its canonical sub-schema with [`ajv-cli`](https://github.com/ajv-validator/ajv-cli), run from the repo root:

```bash
for d in vectors/*/v0.1/cases.json; do
  npx -y ajv-cli@5 validate \
    -s schema/v0.1.json \
    -r "schema/v0.1/*.json" \
    -d "$d" \
    --spec=draft2020
done
```

Both shapes above are load-bearing. The sub-schemas go in as **one quoted glob**,
which `ajv` expands itself: `ajv-cli` 5 accepts exactly one `-r`, and rejects a
second outright as `invalid syntax (too many arguments)` — so a flag-per-file
loop reports every category invalid and reads like broken vectors rather than a
broken command. And each category is validated on **its own run**, because a
combined all-`-d` run has tripped a false negative through a schema `oneOf`
artefact.

Inspect a single case with `jq`:

```bash
jq '.cases[0]' vectors/hkdf-sha256/v0.1/cases.json
```

Every `cases.json` must validate clean. Byte-level conformance against the live reference implementation is verified separately, from the main repo:

```bash
cargo test -p protocol --test vectors                          # most categories
cargo test -p myne-desktop --lib backup_container_vectors      # backup-container (std/fs-bound)
```

The TypeScript-runner categories (`property-coercion`, `markdown-comment`, `d1-clip-serializer`) are asserted by the main repo's Vitest suite against the live dialect code.

## Cross-implementation discipline

Independent ports verify compatibility by running these vectors through their own language's runner. **A failed vector in any port is a stop-the-world bug** — either the port has drifted or the spec is wrong. Do not regenerate vectors to "fix" a failing port without first understanding why it failed. Bit-identical means byte-identical: endianness, padding, and reserved bytes all matter, so cross-language tests should diff bytes, not parsed structures. A shipped `vector_id` is part of the contract and is never reused.

## License

[CC-BY-4.0](LICENSE) — Creative Commons Attribution 4.0. These vectors are reference material for independent implementations and should be maximally reusable as long as Myne is credited. (The wider Myne project keeps its `docs/` specs under the same CC-BY-4.0 content license; application code is AGPL-3.0. This repo carries data only.)
