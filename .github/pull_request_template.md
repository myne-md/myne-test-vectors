<!--
PR title = conventional-commit subject carrying the Plane ID, e.g.
  feat(vectors): land the graph-layout MYG1 category (MYND-###)
Do NOT mention AI/assistant tooling anywhere — the commit-msg hook rejects attribution trailers.
-->

## Summary

<!-- What does this change, and why? -->

## Linked work

- Plane task: MYND-
- Closes #

## Type of change

- [ ] New vector category
- [ ] New / changed cases in an existing category
- [ ] Schema fix
- [ ] Documentation

## Checklist

- [ ] Conventional-commit subject carrying the Plane ID.
- [ ] Vectors were emitted by their generator in the main repo, not hand-edited (unless the category is hand-curated, e.g. `phrase_normalize` / `property-coercion`).
- [ ] `vector_id`s are new and stable — no existing id was renamed or reused.
- [ ] Hex is lowercase with no `0x` prefix; inputs and expected outputs are byte-exact.
- [ ] Every `cases.json` validates against its sub-schema (`ajv-cli validate`, see README).
- [ ] For a new category, the invariant holds: category dir + `schema/v0.1/<cat>.json` + root-schema `enum` entry + root-schema `oneOf` branch all added in lockstep.

## Developer Certificate of Origin

- [ ] I certify the [DCO](https://developercertificate.org/): this contribution is mine to give under the repository's license (CC-BY-4.0), and I understand these vectors are meant to be reusable by any independent implementation with attribution to Myne.
