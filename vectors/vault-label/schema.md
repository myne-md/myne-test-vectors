# `vault-label` vector schema

Pins the byte mapping of `protocol::vault_label_blob::encode_vault_label_blob_raw` + `protocol::vault_label_blob::decode_vault_label_blob` (Rust crate `protocol`, in the [main `myne` repo](https://github.com/myne-md/myne)). The codec emits and consumes `vault.label` at the vault root, the v0.1 **sealed label store** (`MYL1` magic; myne-md/myne#411), which is the device-local cache that lets the vault picker show a vault's name **before** any unlock.

**Reference implementation:** `protocol/src/vault_label_blob.rs::{encode_vault_label_blob_raw, decode_vault_label_blob, parse_vault_label_header}` and `protocol/src/aead.rs::aad_vault_label_blob`.
**Spec:** `docs/spec/protocol.md` § 9.22 (layout, per-vault key, AAD, size cap, erase) + § 5.2 AAD framing + § 6.4 non-distinguishing failures + § 10.3.1 strict-trailing.

## Why the family exists

A vault's authoritative name is the `label` field of `vault.prefs` (§ 9.5.2), sealed under a `root_key`-derived KEK, so it is readable only **after** that vault is unlocked. The picker runs before any unlock, so without this cache it can only draw a neutral fallback. `vault.label` carries the name and nothing else, one file per vault, beside `vault.quickunlock`.

A vault this device has never unlocked simply has no `vault.label` file, which is the correct state rather than a gap: there is no key with which to read a name the device has never held.

## `MYL1` magic allocation

The 4-byte magic `MYL1` (`4d 59 4c 31`) is a free-pool § 10.3 allocation (`L` = label, myne-md/myne#411), implementer-chosen the same way every family's magic is (cf. `MYF1`'s "`F` = file" row). The vectors are generated with `MYL1`; a later letter change would regenerate this category's `expected_*_hex` bytes and the layout below in lockstep. A `vector_id` is never reused, so a magic change is a fresh case set rather than an edit.

## No HKDF subkey role, and the key is per vault

Every § 9 **vault** blob seals under `DeriveSubkey(root_key, …)`. `vault.label` does not, for two reasons that stack:

- the label must be readable while the vault is **locked**, so `root_key` does not exist at read time;
- a derived subkey cannot be destroyed on its own while its root survives, so it could only *look* like the per-`VaultId` erase this family rests on.

So the blob is sealed under a plain 32-byte `wrap_key` held in the OS keychain (service `md.myne.vaultlabel`, distinct from `md.myne.quickunlock`, `md.myne.captures` and `md.myne.shareimport`; non-biometric, device-only). There is **no `hkdf-sha256` role** for vault-label (do not look for one), and the vectors supply `wrap_key` **directly** as a fixed fixture.

One further difference from the two device-local staging queues (§ 9.18, § 9.20): they mint **one key per device**, because an entry is enqueued before anyone knows which vault it lands in. Here the key is **one per vault**, because vault deletion carries a per-`VaultId` cryptographic-erase guarantee, and a shared key would leave a deleted vault's name readable at the one artefact a person actually recognises.

## The AAD binds the `VaultId` the reader expected

This is the property that separates the family from its siblings, and the reason two of the cases exist.

```text
AAD_vault_label_blob =
  "myne.v0.1|" (10 B) || 0x01 (1 B) || u16_be(16) || "vault-label-blob" (16 B) || vault_id (16 B)
```

45 bytes. Neither `account_number_hash` nor `items_key_id` is bound: the label sits outside every `root_key` lineage, and one account can hold two vault directories on one device.

§ 9.18.3 and § 9.20.3 both record a limit they cannot close: their AAD binds the id **the blob carries**, so a sealed entry copied to a second filename is unaltered and still opens, leaving agreement to a reader-side check. Here the caller already knows which vault it is reading (the id is the directory it is reading from), so the **expected** id goes into the AAD and the header's copy is compared against it first. That splits the cross-vault surface in two, and a port has to get both halves right:

| What happened on disk | Which check fires | Case |
|---|---|---|
| Vault A's file turns up where B's should be, untouched | structural comparison, before any key is touched | `parse-fail-vault-id-mismatch` (`vault_id_mismatch`) |
| The same file with its header rewritten to claim B | the AAD, since the comparison now passes | `decrypt-relabelled-header-vault-id` (`aead_fail`) |

The two MUST surface different outcomes. A port that only compares, and never binds, passes the first and fails the second; a port that only binds, and never compares, passes the second and reports the wrong error for the first.

The AAD role string `"vault-label-blob"` (16 ASCII bytes, length-prefixed) is the only load-bearing role string here. Like `aad_capture_blob` and `aad_share_import_blob`, this device-local builder is deliberately **not** enrolled in the vault-scoped `ctx-aead` / `aad-construction` builder-enumeration categories, so its canonical bytes are pinned inside this family instead, by the `aad_construction` case and by every `encrypt` case.

## Size cap, and the empty label

`ciphertext_len` is variable (`plaintext_len + 32`) with a **4 KiB plaintext cap** (§ 9.22.4), and the cap is enforced from both directions:

- **read side:** `ciphertext_len > 4096 + 32` is rejected **before allocating**, so a hostile length cannot size an allocation (`parse-fail-ciphertext-len-overflow`);
- **write side:** an over-cap label is **refused outright rather than truncated**, because a silently shortened name is a wrong name (`encode-fail-over-cap`). The boundary is inclusive, which is why `encrypt-label-at-cap` (exactly 4096 bytes) is a case: a port that writes `>=` where the spec says `>` fails that one and passes everything else.

**An empty label is never stored.** § 9.22.4 makes "no file" the single representation of "this device has no name for that vault", so a host MUST NOT write a zero-length plaintext. That is a host-layer rule rather than a codec reject: `encode_vault_label_blob_raw` would happily seal zero bytes. No case encodes one, deliberately, so that this vector set cannot be read as blessing a second representation.

## Strict-trailing is enforced

§ 10.3.1: the blob's total length must equal `header + ciphertext_len` **exactly**. The older device-local families accept a trailing tail; a new family has no compatibility reason to, and the surplus bytes sit outside both the AEAD tag and the AAD, so tolerating them would mean carrying unauthenticated bytes through a successful decode. A blob that is short is `ciphertext_truncated`; a blob that is long is `trailing_bytes`. They are separate errors rather than one length complaint.

## Layout (as fixed by the reference implementation)

```text
Offset  Size  Field             Notes
0       4     magic             ASCII "MYL1"
4       1     protocol_version  0x01
5       16    vault_id          the UUIDv7 naming vaults/<id>/; AAD-bound
21      24    wrap_nonce        XChaCha20-Poly1305 nonce
45      4     ciphertext_len    big-endian uint32, = plaintext_len + 32
49      ..    ciphertext        UTF-8 label AEAD-sealed + 32-byte CTX tag
```

Header = 49 bytes, the § 9.18.1 / § 9.20.3 shape with `vault_id` where those carry a per-entry id. Magic, `protocol_version`, the `vault_id` binding, `ciphertext_len` and the strict-trailing equality are all validated **before** any decryption is attempted.

## Case kinds

- **`aad_construction`** is the 45 canonical AAD bytes for a vault id. 1 case.
- **`encrypt`** is the encoder's byte output for a given `wrap_key` + `vault_id` + label + `nonce`. 3 cases: short ASCII, a multibyte label (2-byte, 3-byte and 4-byte UTF-8 sequences, so a port that measures the label in characters rather than bytes fails), and one at exactly the 4096-byte cap.
- **`encode_fail`** is the write-side cap refusal at 4097 bytes. 1 case.
- **`decrypt_ok`** round-trips a blob back to its label bytes under the matching `wrap_key` and the expected `vault_id`. 2 cases (ASCII + multibyte, the latter catching a decoder that re-encodes or lossily converts on read).
- **`decrypt_aead_fail`** is the non-distinguishing `CannotDecrypt` surface (§ 6.4): CTX-tag tamper, the relabelled header described above, wrong `wrap_key`. 3 cases.
- **`parse_fail`** is a distinguishable pre-decryption structural reject, one per `VaultLabelBlobError` decode variant: `magic_mismatch` (twice, see below), `protocol_version_mismatch`, `vault_id_mismatch`, `ciphertext_len_overflow`, `header_too_short`, `ciphertext_truncated`, `trailing_bytes`. 8 cases.

18 cases total.

**Every `parse_fail` fixture carries the REAL `wrap_key`.** None of the corrupted header fields is AAD-bound, so a port that skips one of these checks decodes the fixture **successfully** and the runner catches it. A dummy key would mask exactly that bug behind a coincidental AEAD failure. Three of the fixtures are only useful for that reason: the foreign-magic, wrong-version and trailing-byte blobs each return the label `Work` in a port that omits the corresponding check.

**The magic is checked before decryption, and one case is built to prove it.** `parse-fail-magic-before-decrypt` corrupts the magic **and** destroys the tag, together. A decoder that decrypts first has a cannot-decrypt answer available and will give it; only one that checks the magic first can answer `magic_mismatch`. This matters because checking the magic *after* decrypting still "works" on every valid input, so an ordering bug is invisible to any case that corrupts one thing at a time. The second magic case (`parse-fail-foreign-magic-myq1`) is the same blob wearing `MYQ1`, the quick-unlock credential that literally sits beside `vault.label` at the vault root.

## What this does NOT cover

- **The keychain plumbing.** Which OS keychain, which accessibility class, and how a per-vault key is minted and destroyed are host concerns, verified by main-repo integration tests rather than cross-implementation vectors. The codec only ever sees a symmetric `wrap_key`.
- **`RandomnessUnavailable`.** It is raised by the non-`_raw` wrapper when the OS CSPRNG fails, and cannot be expressed as a fixture.
- **The erase path.** Destroying a key, removing the file, and the ordering between them (§ 9.22.5) are host behaviour, not codec bytes.
- **The read-side fallback policy.** § 9.22.4 says every failure collapses to the neutral name on the picker. These vectors pin which error the codec returns; what a client draws on failure is a UI contract.

## Regeneration

The main repo does not yet carry an `emit_*_for_paste` generator or a `tests/vectors.rs` runner for this family. The bytes in `v0.1/cases.json` were produced by, and re-verified against, the shipping `protocol` crate through an out-of-tree harness that links it by path, so they are the reference implementation's own output. Landing the in-repo generator and runner (the usual four edits, including the `mod embedded` row for the browser lane) is follow-up work in the main repository.
