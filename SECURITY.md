# Security Policy

## Reporting a vulnerability

Report security issues to **security@myne.md** privately. Please do not open a
public GitHub issue for a vulnerability.

If you do not get an acknowledgment within 5 business days, send a reminder;
the address may have failed silently.

## What to include

- A description of the issue and its impact.
- Steps to reproduce, or a proof of concept.
- The affected component and version/commit (this repository, the main
  `myne-md/myne` repository, or `myne-md/myne-server`).
- Any suggested mitigation.
- How you want to be credited.

## Response commitment

- Acknowledge within 5 business days.
- Initial assessment within 14 days.
- Coordinated disclosure: we ask for a 90-day window from acknowledgment,
  extended by agreement if a fix needs more time.

## Scope

In scope: the vectors and schemas in this repository, the protocol
specification they pin, and the reference clients and server under
`myne-md/`. A vector that is wrong, ambiguous or internally inconsistent is a
security-relevant report: independent implementations will disagree on the
wire.

Out of scope: third-party dependencies (report upstream first), and
self-hosted instances run by third parties (report to the operator).

## Pre-1.0 caveat

Myne is pre-alpha. The protocol is v0.1 under active hardening and has not had
a third-party cryptographic audit. Reports outside the published threat model
are still welcome, but may be classified as known limitations.

## Encrypted reports

We do not publish a PGP key. If your report contains material you do not want
to send by plain email, contact security@myne.md first to arrange a transport.
