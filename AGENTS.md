# AGENTS.md

Guidance for coding agents working in this repository. It is public: do not add
private hostnames, credentials, internal runbook names or links to repositories
that are not published.

## What this repository is

Language-agnostic JSON test vectors that pin the Myne cryptographic protocol at
the byte level: v0.1 in full, plus the parts of v0.2 that are already frozen.
There is no build step and no code to compile. The contract is the data plus the
JSON Schemas; a port verifies itself by running its own language's test runner
against this repository.

## Layout

- `schema/<version>.json` — root schema; dispatches to the per-category
  sub-schemas by `schema` through `oneOf`.
- `schema/<version>/<category>.json` — one sub-schema per category.
- `vectors/<category>/schema.md` — domain-semantic description of the category.
- `vectors/<category>/<version>/cases.json` — the vectors themselves.
- `lefthook.yml` — commit-time JSON/text hygiene and a local `ajv` sweep.
- `.github/workflows/validate.yml` — the same sweep in CI.

## Validate

```bash
npm install --global ajv-cli@5
for d in vectors/*/v*/cases.json; do
  ver=$(basename "$(dirname "$d")")
  ajv validate -s "schema/$ver.json" -r "schema/$ver/*.json" -d "$d" --spec=draft2020
done
```

Run each category on its own. A combined all-`-d` run has historically produced
a false negative through a `oneOf` artefact.

## Rules

- Data, schemas and documentation are CC-BY-4.0. Scripts and CI configuration
  are MIT. Banners and Myne marks are not licensed. See `LICENSING.md`.
- The invariant, **per version**: category directories = sub-schema files =
  root-schema `enum` entries = root-schema `oneOf` branches. All four move
  together or the landing is incomplete.
- A shipped `vector_id` is part of the contract; never rename or reuse it.
- Where a generator exists in the main repo, do not hand-edit the output.
  Hand-curated categories are named in their `schema.md`.
- Hex is lowercase with no `0x` prefix. Inputs and expected outputs are
  byte-exact.
- Commits carry a `Signed-off-by:` trailer (DCO 1.1). The project does not use
  a CLA.
- Never commit credentials, tokens, private keys or internal hostnames.
- Security reports go through `SECURITY.md`; do not open a public issue.
