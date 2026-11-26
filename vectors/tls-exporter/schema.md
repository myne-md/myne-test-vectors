# `tls-exporter` vector schema

Pins the two frozen [RFC 9266](https://www.rfc-editor.org/rfc/rfc9266.html) TLS
keying-material **exporter parameters** that generate the channel-binding data
every Myne SCRAM-SHA-256-**PLUS** handshake binds to.

**Reference implementations:**
`core/src/plugins/sync/enroll.rs` (`TLS_EXPORTER_LABEL` / `TLS_EXPORTER_LEN`) in
the [main `myne` repo](https://github.com/myne-md/myne), and
`internal/auth/scram/scram.go` (`ExporterLabel` / `ExporterLen`) in
the `myne-server` companion repository.
**Spec:** RFC 9266 §3 (`tls-exporter` channel binding for TLS 1.3) ·
`docs/spec/server-protocol.md` §3.1 (SCRAM-SHA-256-PLUS).

## Why a category exists for two constants

This is the smallest category in the repository — one case, two values — and the
reason it is here is not completeness for its own sake.

The label and the length are a **cross-repository agreement between two
languages**. The Rust client asks its TLS stack for exporter output under a
label; the Go server asks its own TLS stack for the same thing; SCRAM then binds
the result into the client proof. If the two sides disagree on either value by a
single byte, both compute a well-formed proof over *different* `cbind_data`, the
server's verification fails, and **no `-PLUS` handshake can ever complete** — a
total authentication break rather than a degraded mode, and one that reproduces
identically on every device, so it reads like a credential bug rather than a
constant mismatch.

Until this category landed, the two constants were hand-written copies in two
repositories, each carrying a comment saying the other must match it
byte-for-byte, and neither checked against anything. A comment is not a contract.

## What this category does NOT pin

**The exporter itself.** The `tls-exporter` computation is the TLS library's
(RFC 8446 §7.5 key schedule); Myne neither implements nor re-derives it, so there
is no derivation here to run and no expected output to compare. What a port must
reproduce is the *inputs it hands its TLS stack*, and that is what the case
carries.

The SCRAM arithmetic that consumes `cbind_data` — `Client Key`, `Server Key`, the
client proof, the server signature — is pinned separately and independently by
the [`scram-auth`](../scram-auth/schema.md) category, which is deliberately
agnostic about where the binding bytes came from. The two categories meet only in
production.

## File layout

```
vectors/tls-exporter/
├── schema.md              ← this file
└── v0.1/
    └── cases.json         ← 1 case (the frozen parameter pair)
```

## JSON shape

The machine-readable contract lives at [`schema/v0.1.json`](../../schema/v0.1.json)
(top-level shape) and [`schema/v0.1/tls-exporter.json`](../../schema/v0.1/tls-exporter.json)
(per-case shape). One `case_kind`:

- **`parameters`** — the frozen input pair. Fields:
  - `expected_label_utf8_hex` — the exporter label as **hex-encoded UTF-8**, not
    a bare string. This is the one shape decision in the category worth stating:
    the label is compared byte-for-byte deep inside a TLS stack, and a trailing
    space, an inserted NUL or a Cyrillic homoglyph would survive an editor
    round-trip, a code review and a JSON diff completely unnoticed while breaking
    every handshake. Hex makes those edits visible.
  - `informational_label_utf8` — the human-readable rendering of the same bytes,
    so a reviewer does not have to decode hex to read the file. **Informational:
    the hex is normative**, and a port that reads this field instead is testing
    its own JSON parser.
  - `expected_length` — the exporter output length in bytes (RFC 9266 uses 32).

## How to consume

A port asserts these values against **its own constants**, not against a computed
result:

1. Hex-decode `expected_label_utf8_hex`; compare byte-for-byte with the label
   string the port passes to its TLS exporter API.
2. Compare `expected_length` with the output length the port requests.

There is no crypto to run. That is the point — the failure this category prevents
is a *typo*, and a typo needs an equality check rather than a KAT.

## Invariants the vectors pin

- **The label is `EXPORTER-Channel-Binding`** (24 ASCII bytes), the RFC 9266 §3
  value verbatim. Not a Myne-chosen string: an implementation that invents its
  own label is not doing RFC 9266 channel binding, whatever else it is doing.
- **The output length is 32 bytes.** RFC 9266 fixes this; a port requesting a
  different length derives different `cbind_data` from an otherwise identical
  connection.
- **Both sides of the wire read the same file.** The Rust client and the Go
  server are each checked against this one published contract rather than against
  a comment in the other's repository — which is the entire reason the category
  is not simply a unit test in one of them.

## What this does NOT cover

- **The `tls-exporter` derivation** (the TLS library's, per RFC 8446 §7.5) — see
  above.
- **The SCRAM proof arithmetic** over `cbind_data` — the
  [`scram-auth`](../scram-auth/schema.md) category.
- **The `gs2-cbind-flag` / channel-binding negotiation**, including the
  empty-`cbind` fallback a proxy-terminated or loopback-HTTP deployment takes
  (`enroll.rs` binds to empty when the server does not signal `-PLUS`). That is
  connection-level negotiation, not a frozen constant.
- **TLS versions other than 1.3.** RFC 9266's `tls-exporter` binding is defined
  for TLS 1.3; Myne's transport requires it.

## Regeneration

There is no generator. Both values are frozen constants transcribed from RFC 9266
§3 — regenerating them from an implementation would defeat the purpose, since the
implementation is precisely what the case exists to check. Changing either value
is a protocol-compatibility event that invalidates every deployed pairing between
a client and a server, and belongs in `design notes` before it belongs here.
