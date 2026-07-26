# `scram-auth` vectors (v0.1)

Byte-level KAT for the **SCRAM-SHA-256-PLUS client proof** — the deterministic
client-side crypto of the Phase 2 device-authentication handshake. Spec
`docs/spec/server-protocol.md` (auth, MYND-420) · RFC 5802 / 7677 / 9266.
Reference implementation: `protocol/src/scram.rs::scram_client_proof`.

The secret being proven is `DeriveSubkey(root_key, SubkeyRole::ServerAuth)` —
HKDF role `"server-auth"`, pinned independently in the
[`hkdf-sha256`](../hkdf-sha256/schema.md) category. It is **never transmitted**:
the server stores only the `StoredKey` / `ServerKey` verifier pair, so a server
breach yields no credential that can authenticate elsewhere, and the zero-
knowledge posture survives a hostile host (TB5).

**`-PLUS` means channel binding** (RFC 9266): the proof commits to the TLS
`tls-exporter` value, so a proof captured on one TLS connection cannot be
replayed on another. The GS2 header is `"p=tls-exporter,,"`.

## Derivation

```
SaltedPassword  = PBKDF2-HMAC-SHA256(secret, salt, iterations)      (32 B)
ClientKey       = HMAC(SaltedPassword, "Client Key")
StoredKey       = SHA-256(ClientKey)                       ← server-stored
ServerKey       = HMAC(SaltedPassword, "Server Key")       ← server-stored

cbind_input     = base64("p=tls-exporter,," || cbind_data)
AuthMessage     = "n=" username ",r=" client_nonce
               || ",r=" combined_nonce ",s=" base64(salt) ",i=" iterations
               || ",c=" cbind_input ",r=" combined_nonce

ClientSignature = HMAC(StoredKey, AuthMessage)
ClientProof     = ClientKey XOR ClientSignature            ← sent by client
ServerSignature = HMAC(ServerKey, AuthMessage)             ← checked by client
```

**Closed-stack note:** the only KDF is `pbkdf2` (RFC-mandated; RustCrypto crate,
never hand-rolled). Every HMAC-SHA-256 is `Hkdf::<Sha256>::extract` (RFC 5869
§2.2) — the same construction as `compute_vault_head_mac`, so no `hmac`-crate
surface enters beyond what `pbkdf2` already pulls. This is why the Myne stack can
speak standard SCRAM without adding a primitive to the audited set.

**`iterations` is a wire parameter, not a security floor here.** The vectors use
`iterations = 1` deliberately so the KAT runs instantly; the deployed floor lives
in the server protocol, not in this math. A port must not "helpfully" clamp it —
that changes `SaltedPassword` and every downstream value.

## Cases (3)

All carry `case_kind: "scram_plus"`. Each fixes the full handshake parameter set
(`input_secret_hex`, `input_salt_hex`, `input_iterations`, `input_username`,
`input_client_nonce`, `input_combined_nonce`, `input_cbind_data_hex`) and pins
all four outputs (`expected_stored_key_hex`, `expected_server_key_hex`,
`expected_client_proof_hex`, `expected_server_signature_hex`).

- **`basic`** — the reference handshake.
- **`zero-secret`** — an all-zero `secret`, the classic edge probe that catches
  an implementation which skips the PBKDF2 stage or treats an all-zero key as
  "no key".
- **`alt-cbind`** — identical to `basic` except for `input_cbind_data_hex`. The
  pair pins that the channel-binding data actually reaches `AuthMessage`: a port
  that ignores it produces `basic`'s proof for both cases and fails here. This is
  the case that makes it `-PLUS` rather than plain SCRAM.

Pinning `StoredKey` and `ServerKey` alongside the proof means a port whose proof
is wrong fails at the layer where it diverged — PBKDF2, the key derivation, or
the `AuthMessage` assembly — instead of one opaque mismatch at the end.

## How to consume

Decode the `input_*` fields, run the port's `scram_client_proof` equivalent, and
compare all four expected values byte-for-byte. Nonces and username are UTF-8
strings, not hex; `cbind_data` and `secret` / `salt` are hex.

Any mismatch is a stop-the-world bug (per repo `README.md` "Cross-implementation
discipline"). These vectors are the Rust ↔ Go bit-identity contract: the same
file drives `myne-server`'s `internal/auth/scram`.

## R3 caveat (READ THIS)

`protocol/src/scram.rs` is the **deterministic math**, and that is what these
vectors verify. The live `tls-exporter` wiring over the rustls layer
(`src-tauri`) and the channel-binding assembly against a real handshake are
**R3-gated** — they still owe a primitive-level review plus a live interop test
before production trust (mirroring the Go server's
`review-r3-scram-tls-exporter.md`). A green `scram-auth` run means the arithmetic
agrees, not that the transport is reviewed.

## What this does NOT cover

- **`server-auth` secret derivation** — [`hkdf-sha256`](../hkdf-sha256/schema.md)
  (`zero-root-server-auth` / `prn-alpha-server-auth`).
- **The SCRAM wire messages.** `scram_client_final_message` assembles
  `c=…,r=…,p=…`; its shape is exercised by in-crate unit tests, not vectored.
- **Server-side verification, nonce generation, iteration-count policy, and
  replay windows** — server behavior in `server-protocol.md`.
- **Plain (non-`-PLUS`) SCRAM**, the documented fallback if `-PLUS` slips: not
  wired in this module, so not vectored.

## Regeneration

There is no `#[ignore]`-gated generator for this category; the cases were
authored against the RFC test-vector methodology and are asserted by the runner
`scram_auth_vectors` in `protocol/tests/vectors.rs` (skip-if-missing). Changing
any expected value here is a cross-implementation break that must be reconciled
with the Go server in the same change.
