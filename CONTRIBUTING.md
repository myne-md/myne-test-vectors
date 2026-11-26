# Contributing

Thanks for helping make the vectors a better cross-implementation contract.
This repository is data and schemas; changes are judged by byte-exactness and
by whether an independent port can reproduce the case without reading the Rust
reference implementation.

## Before you start

- For a new category or a missing case, open an issue first using the
  **Coverage gap / new vector request** template. A new category also needs a
  runner in the main `myne-md/myne` repository, so coordinate there.
- For a disagreement between your implementation and a published vector, use
  the **Vector disagreement** template. A failed vector is a stop-the-world
  bug: do not regenerate the vector to make a port pass until you know which
  side is wrong.

## What a change must contain

- `vectors/<category>/<version>/cases.json` with a stable, new `vector_id`.
- The category's `schema.md` updated when the domain semantics change.
- For a new category: `schema/<version>/<category>.json`, plus the matching
  root-schema `schema` enum entry and `oneOf` branch **in the same change**.
  The invariant is per version: category dirs = sub-schema files = enum =
  `oneOf`.
- Lowercase hex, no `0x` prefix, byte-exact inputs and outputs.
- Cases emitted by the main repo's generator, not hand-edited, unless the
  category is explicitly hand-curated.

## Validate locally

```bash
npm install --global ajv-cli@5
for d in vectors/*/v*/cases.json; do
  ver=$(basename "$(dirname "$d")")
  ajv validate -s "schema/$ver.json" -r "schema/$ver/*.json" -d "$d" --spec=draft2020
done
```

CI runs the same loop plus the category/schema/enum/`oneOf` invariant check.

## Developer Certificate of Origin

Every commit must carry a `Signed-off-by:` trailer for its author:

```bash
git commit -s
```

The sign-off certifies the [Developer Certificate of Origin 1.1](https://developercertificate.org/).
Contributions are accepted under the repository's licenses: data, schemas and
documentation under CC-BY-4.0; scripts and CI configuration under MIT. There is
no CLA.

## Style

- Conventional-commit subjects, for example
  `feat(vectors): pin the graph-layout MYG1 category`.
- Keep a PR focused: one category or one coherent case family.
- Explain the protocol surface and the spec section in the PR summary.

## Security

Do not open a public issue for a vulnerability. Follow `SECURITY.md`.
