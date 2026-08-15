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
- `expected_emit_alias`: the alias the emitter must produce from this parse, byte for byte.
  It carries the RETENTION half of the rule, which nothing else here pins. For an `unknown-key`
  case the three fields above are byte-identical whether the parser retains the unrecognised
  segment or throws it away, so retention was observable only through `expected_export_suffix`,
  which the **Rust** runner reads. A TypeScript parser that dropped `future=keepme` therefore
  passed every case in this file, and the next time the author touched that embed the emitter
  rebuilt the alias without it and the bytes were gone. Both retained buckets belong here: an
  unrecognised key, and a known key whose value this version cannot read. The second is what
  version skew produces the moment a newer client widens a bound, and the two differ in whether
  the reader is TOLD, not in whether the bytes survive.

## Consumers

- **TypeScript**, in the main repo's Vitest suite: asserts `parseEmbedAlias` against
  `expected_filename` / `expected_layout` / `expected_dropped`, and `emitEmbedAlias` against
  `expected_emit_alias`.
- **Rust**, in `myne-core`: asserts `obsidian_layout_suffix` against `expected_export_suffix`.

A case that fails on either side is a stop-the-world bug: it means a note laid out on one client
reads differently on another, or an export silently loses a choice the author made.

## What this does NOT cover

Stated rather than left to be discovered, because a conformance category's silence reads as
coverage.

- **Four ambiguous parses have no case**, and a reader building from the prose here has to guess
  the same way: an unkeyed segment AFTER a keyed one (`photo.png|w=300|foo.png`), a trailing pipe
  or empty segment (`photo.png|`), whitespace around a key (`photo.png| w=300`, `w = 300`), and an
  empty filename carrying layout (`|w=300`). A hand-typed alias will have the third.
- **The `h=` semantic is one word in the grammar and two behaviours in the renderer.** An image
  applies it as an exact box; a PDF applies it as a maximum height. The value grammar is identical
  and no case distinguishes them, because these vectors pin the PARSE, not the render.
- **Nothing here exercises a `]` or a line break inside an alias.** Both are forbidden by
  `wikilink.md` §10 rather than by this grammar, and both are refused at the authoring path.

## Regeneration

Hand-curated, no generator. `expected_emit_alias` is the one field mechanically derivable from the
others, and it is written out rather than derived on purpose: a field the runner computes the same
way the implementation does asserts nothing.
