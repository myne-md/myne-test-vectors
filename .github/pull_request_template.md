## Summary

<!-- What does this change, and why? -->

## Linked work

- Closes #

## Type of change

- [ ] New vector category
- [ ] New / changed cases in an existing category
- [ ] Schema fix
- [ ] Documentation

## Checklist

- [ ] The change is byte-exact: lowercase hex, no `0x` prefix.
- [ ] `vector_id`s are new and stable — no existing id was renamed or reused.
- [ ] Vectors were emitted by their generator in the main repo, not hand-edited (unless the category is explicitly hand-curated).
- [ ] Every `cases.json` validates against its sub-schema (see README).
- [ ] For a new category, the invariant holds: category dir + `schema/<version>/<cat>.json` + root-schema `enum` entry + root-schema `oneOf` branch all added in lockstep.

## Developer Certificate of Origin

- [ ] I certify the [DCO](https://developercertificate.org/): this contribution is mine to give under the repository's license (CC-BY-4.0 for data, MIT for code), and I understand these vectors are meant to be reusable by any independent implementation with attribution to Myne.
