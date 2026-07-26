# `head-fingerprint` vectors (v0.1)

Byte-level KAT for the **head SAS fingerprint** — the short human-comparable
string derived from a `head_id`. Spec `docs/spec/sync-protocol.md` **§6.3**
(SYNC-17). Reference implementation: `protocol/src/vault_head.rs::head_fingerprint`.

This is the value two devices **eye-compare** out-of-band: at device enrollment
(MYND-425) and during a fork audit (MYND-427). If the two humans read different
strings, the server has forked or substituted the vault head. Its whole security
value rests on both devices computing it identically from the same `head_id`,
which is what these vectors pin.

## Derivation

```
digest = SHA-256("myne.v0.1 vault-head-fingerprint" || head_id)
acc    = first 5 bytes of digest, big-endian            (40 bits)
chars  = 8 × 5-bit groups of acc, most-significant first
         each group indexes the frozen Crockford base32 alphabet
         "0123456789ABCDEFGHJKMNPQRSTVWXYZ"
output = "XXXX-XXXX"   (a dash after the 4th character)
```

Three details are load-bearing and each is a place a port can silently drift:

1. **The domain string** `"myne.v0.1 vault-head-fingerprint"` (32 ASCII bytes,
   note the space after `myne.v0.1`, matching the HKDF prefix convention rather
   than the `|`-terminated AAD convention). It is a SHA-256 **domain separator**,
   not an HKDF info — the fingerprint is a hash, not a derived key.
2. **40 bits, not 40 characters.** Only the first 5 bytes of the digest feed the
   encoding; a port that base32-encodes the whole digest and truncates the
   *string* produces different characters, because the 5-bit grouping no longer
   lines up with the same bits.
3. **The Crockford alphabet is the same frozen table the account number uses**
   (`account-number.md` §4.1) — no `I`, `L`, `O`, `U`. The dash is presentation,
   inserted after the 4th character; it is not part of any hashed input.

40 bits is a deliberate SAS trade-off: short enough that a human will actually
compare it, long enough that a server forging a matching fingerprint needs ~2^40
work per attempt against a target the user re-checks.

## Cases (4)

Single-shape category — no `case_kind` discriminator. Each case maps one
`input_head_id_hex` (32 bytes) to one `expected_fingerprint` string:

| `vector_id` | `head_id` | fingerprint |
|---|---|---|
| `all-zero` | 32 × `0x00` | `TYRS-N318` |
| `all-0x42` | 32 × `0x42` | `W7P9-W8NH` |
| `all-0xff` | 32 × `0xFF` | `8BZ1-15PW` |
| `ascending` | `00 01 02 … 1f` | `1056-DP8K` |

The all-zero and all-`0xFF` inputs are the classic edge probes (they catch
index-by-zero and sign-extension defects that a random input would mask); the
ascending case catches byte-order and grouping errors, since every input byte is
distinct.

## How to consume

Decode `input_head_id_hex`, run the port's `head_fingerprint` equivalent, and
compare the resulting string **exactly** — including the dash and the uppercase
letters. Any mismatch is a stop-the-world bug (per repo `README.md`
"Cross-implementation discipline"): a fingerprint that disagrees across
implementations makes the whole out-of-band comparison meaningless.

## What this does NOT cover

- **How `head_id` is computed** — [`vault-head`](../vault-head/schema.md) (it is
  `SHA-256(head_mac_input ∥ head_mac)`, §3.4 gap G3).
- **The enrollment / fork-audit UX** that shows the string — client behavior in
  `sync-protocol.md` §6.3, not a cross-implementation contract.
- **Crockford decoding or checksum behavior** — the fingerprint is display-only;
  it is never parsed back. The Damm-32 check character belongs to the account
  number ([`damm32-table`](../damm32-table/schema.md)), not here.

## Regeneration

These cases are emitted by running `protocol::head_fingerprint` over the listed
`head_id`s. There is no dedicated `#[ignore]`-gated generator: the function takes
one input and returns one string, so the four cases are the table above.

Runner: `head_fingerprint_vectors` in `protocol/tests/vectors.rs`
(skip-if-missing).
