# `embed-layout` vector schema

Pins the **form-C embed layout segments** (`docs/spec/wikilink.md` §2 form 12 / §5.1 and
`docs/spec/markdown-dialect.md` §6 / §7, myne-md/myne#409, in the
[main `myne` repo](https://github.com/myne-md/myne)): how a conformant client splits an attachment
embed's alias into a display filename plus the author's layout choices, and what the
attachments-included export writes in their place.

```
![[att:<attachment-id-hex>|photo.png|w=300|align=center|alt=A sunset | 2026]]

w     = <1..8192> | <1..100>%
h     = <1..8192>
align = left | center | right
alt   = terminal, swallows the rest
```

**Reference implementation:** split TS-side and Rust-side, which is exactly why this category
exists. The canonical grammar is `@myne/dialect/embedLayout` (TypeScript, source-consumed by the
desktop, web and mobile clients); the export transform is `myne-core`'s
`rewrite_embeds_to_final_names` (Rust, shared by the desktop and the web wasm entry crate). Neither
side can detect drift from the other alone.

## What this category is, and is not

A **deterministic-behavior conformance** category in the `property-coercion` / `markdown-comment`
class: not an on-disk codec, not cryptographic. Nothing here is a magic byte, an AAD role or a
subkey context. What it guards is agreement between two implementations of one text grammar, plus
four rules that each exist because of a specific failure:

- **Segment 1 is always the filename, and an unkeyed later segment rejoins it.** An attachment
  filename can already contain a `|` today: nothing rejects one on rename, and a pipe is a legal
  filename character on macOS and Linux, so `![[att:<id>|re|port.png]]` is already on disk. A
  positional or shape-recognised grammar would silently reinterpret those notes, reading the
  filename as `re` and `port.png` as a layout segment. This is the load-bearing case
  (`filename-with-pipe`, `filename-with-pipe-and-key`, `filename-looks-like-a-key`).
- **`alt=` is terminal.** Alt text is free prose and contains pipes, and the sanctioned import
  transform carries a markdown image's alt (`![A sunset | 2026](photo.png)`) straight into this
  slot, so forbidding the character would mean mutating user content on import. An authored-EMPTY
  `alt=` is a real value meaning a decorative image, distinct from an absent one
  (`alt-terminal-with-pipes`, `alt-terminal-swallows-later-key`, `alt-authored-empty`).
- **Reporting splits by case.** A **known** key carrying an invalid value is dropped AND reported
  with its source text verbatim; an **unrecognised** key is dropped **silently**. The split is
  normative rather than cosmetic: three clients ship on independent versions against one synced
  vault, so a key an older client does not know is routine, the note is right and the reader can do
  nothing about it. Reporting it would put a warning on every embed a newer client styled
  (`invalid-*` versus `unknown-key-*`). An over-ceiling value is **dropped, not clamped**, because
  clamping silently changes the author's stated intent.
- **The export translates narrowly and never deletes.** A pixel `w=` (optionally with `h=`) becomes
  Obsidian's own `|300` / `|300x200` spelling; everything else, including a percentage width
  Obsidian has no slot for and a value the Rust side cannot read, travels through unchanged in its
  original order after the dimension. Refusing to export bytes one side merely failed to understand
  is how an export loses data (`export` cases, `invalid-width-non-numeric`).

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json) (top-level
shape) and [`schema/v0.1/embed-layout.json`](../../schema/v0.1/embed-layout.json) (per-case shape).
Every case carries the alias as `input_alias_utf8_hex` (plus an `informational_alias` mirror) and
pins the parse and the export. One case shape, discriminated by `case_kind`:

- `case_kind`: `parse`, `filename` (the unkeyed-rejoin rule), `invalid` (known key, rejected
  value: dropped **and** reported), `unknown-key` (outside the closed vocabulary: dropped
  **silently**), `alt-terminal`, or `export` (the case exists mainly to pin the exported suffix).
- `expected_filename`: the display filename after the split. Not the alias.
- `expected_layout`: the resolved choices, every field nullable. `alt` has three states: `null`
  (absent, so the filename is the accessible name), `""` (authored empty, decorative), and text.
- `expected_dropped`: verbatim source text of every segment worth reporting, in document order.
  Empty for every `unknown-key` case, which is the whole point of the split.
- `expected_export_suffix`: what the export appends after the final filename inside
  `![[<final name><suffix>]]`. Empty when there is nothing to carry.

## Consumers

- **TypeScript**, in the main repo's Vitest suite: asserts `parseEmbedAlias` against
  `expected_filename` / `expected_layout` / `expected_dropped`.
- **Rust**, in `myne-core`: asserts `obsidian_layout_suffix` against `expected_export_suffix`.

A case that fails on either side is a stop-the-world bug: it means a note laid out on one client
reads differently on another, or an export silently loses a choice the author made.
