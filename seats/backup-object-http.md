# Backup — Object-HTTP

*Seat implementation page, draft 0.2 — 21 August 2026.*

**Status:** Running in free mode. A person can enrol, back up, and restore
onto a second device from their recovery phrase alone. The paid path is
written but has never met a credential it did not also mint, and the
conformance fixtures are unwritten. See [Honest status](#honest-status).

**Profile:** [`backup/UI-Backup-Object-HTTP.md`](https://github.com/onymchat/onym-system/blob/main/backup/UI-Backup-Object-HTTP.md)
— merged in `onym-system`. It implements the abstract
[`backup/UI-Backup.md`](https://github.com/onymchat/onym-system/blob/main/backup/UI-Backup.md)
boundary: the device seals a whole snapshot under a key derived from
the person's own recovery phrase, addresses it by a digest over the
sealed bytes, and hands an operator opaque chunks over HTTPS. The
operator authenticates a public key, counts bytes, and can do nothing
else with what it holds.

**Operator:** [`onym-backup`](https://github.com/onymchat/onym-backup)
· **Live:** `https://backup.onym.app` (free mode — no entitlement
issuers declared, so it never returns `402`)
· **Client code:** `onym-ios`, `onym-android`
· **Listed in:** the `onym-services` discovery catalog, so a client
finds it the same way it finds every other seat.

This page summarizes what the profile pins, so a reader can judge the
design without the full normative text. Every suite, header, and error
code named below is defined there — as is the required conformance
fixture list, which specifies what an implementation must test, not a
suite that runs today (see [Honest status](#honest-status)).

## What the mapping pins

| Abstract concept | Object-HTTP mapping |
|---|---|
| Operator endpoint | HTTPS origin, operations under `/v1/` |
| Snapshot reference | `sha256:<64 lowercase hex>` over the exact sealed byte sequence |
| Sealing | AES-256-GCM over 1 MiB plaintext chunks under a per-snapshot key |
| Key derivation | HKDF-SHA256 from the holder's BIP-39 seed |
| Access authorization | Request-bound Ed25519 proof of possession, single-use |
| Holder identity at the operator | An Ed25519 public key, and nothing else |
| Increment model | None; whole snapshot, transfer-chunked |
| Payment refusal | HTTP `402` with a `PaymentRequired` body |
| Entitlement | Broker-signed `SeatEntitlement`, verified locally by the operator |

The implementation profile identifier is `onym:backup-implementation:object-http-v1`,
mapping the portable `onym:backup-profile:sealed-device-archive-v1`.

## The root is the recovery seed, not a device key

All key material derives from the holder's BIP-39 seed — the same
mnemonic that recovers identity — through distinct HKDF contexts: an
archive root, a fresh per-snapshot key drawn through a random salt, and
a pair of access keys scoped to the operator's `componentId`. This
sounds obvious and is the single most likely implementation mistake: a
client whose local at-rest encryption uses a device-bound key must
perform a **logical export** — decrypt through its own stores,
re-serialize, and re-seal — not copy its encrypted database files. A
key a device can't export is unrestorable on exactly the device that
will ever need it: the replacement.

Sharing a root with the identity keys has a real cost the profile
states rather than hides: **"lost access key" and "lost identity"
become one event.** The alternative — a separately generated and
separately stored access key — doubles the number of secrets a person
must survive, and the second one exists only for backup, so it's the
one they'll lose. This profile takes the shared root deliberately, and
says a future profile may take the other side of that trade.

Rotation inherits the same cost, one level up: rotating the access key
changes the holder handle the operator recognizes, which orphans every
snapshot retained under the old handle unless the operator supports a
re-binding proof — and this profile doesn't define one yet (see
[Honest status](#honest-status)). A client must therefore either treat
rotation as destructive — erase under the old key first, then start a
fresh archive — or not offer rotation at all, and say so plainly.

## Padding instead of a hard cap

The plaintext archive is padded to a Padmé bucket before sealing:
overhead stays under about 12%, instead of the potential doubling a
power-of-two ladder risks. That matters because storage here is already
strictly linear in holders and can never be deduplicated — the fresh
per-snapshot salt guarantees non-convergent keying, so two people
sealing identical archives get unrelated ciphertext and unrelated
digests. `sealedByteSize` still leaks a coarse shape of a holder's
archive over time; the profile forbids the operator from retaining a
size time series, but nothing on the wire enforces that beyond
declared conduct.

## Proof of possession, not an account

Every request signs method, path, holder, timestamp, nonce, and a
digest of the body — each field length-prefixed so a signature can't be
reinterpreted by shifting a boundary between two attacker-influenced
fields. A holder is an Ed25519 public key; there's no account, no
email, no password, and no recovery question, and the profile makes
that a checkable property rather than a promised one: no route accepts
another kind of credential, no route reassigns a snapshot's holder, and
no administrative route exists at all — not even a token-gated or
localhost-only one.

## Preflight is the entire point of the payment mapping

A `402` has to cost one small request, never a completed
multi-hundred-megabyte upload. Preflight checks terms currency,
entitlement, whether the digest is already retained, size, and quota —
in that order, so a re-check of bytes the operator already holds never
gets refused by a limit that only matters for *new* bytes. **Export
never consults entitlements at all** — not "consult and allow," no
access to entitlement state in that code path — because that's the
only way the abstract contract's promise that a lapsed payment never
holds a person's own history hostage survives future edits to the
implementation.

## Erasure receipts are commitments, not proof

An erasure receipt is a signed acknowledgment measured against the
snapshot's own pinned terms — never proof of destruction. Its
`excludedScope` field is mandatory and must be non-empty: there is
always something an erasure can't reach, starting with the copies held
by every other participant in a conversation the snapshot contained.
Erasure never returns a second, worse acknowledgment either — the
"unconfirmed" state is computed client-side, by comparing the receipt's
committed deadline against the clock, not signaled by the operator.

## Export carries its own terms, forever

The portable export container bundles each snapshot's sealed bytes with
the *bytes* of every terms document it was ever accepted under — not
just a digest and a URL. An operator has to keep serving every
historical terms document it has ever accepted a snapshot under, for as
long as it exists; but a shut-down operator serves nothing, so a
container that only carried a digest would leave a holder, after
shutdown, holding a `termsId` whose preimage is gone — pinning that
survives as an unresolvable reference is not pinning. Migration to a
second conforming operator is an upload of the same bytes under the
same reference: no re-sealing, no cooperation required from the
operator being left.

## Honest status

- **The paid path has never met a real credential.** Nothing issues a
  `SeatEntitlement` — no broker exists in any Onym repository — so
  §10's refusal, purchase, lapse, grace, and revocation behaviour is
  exercised only against credentials the tests mint themselves. The
  deployed operator declares no entitlement issuers and therefore never
  returns `402`, which is the self-hosting path the profile requires
  and not a workaround. Until a broker exists, "the paid path works" is
  a claim about code that has never been paid.
- **The conformance fixtures are still unwritten.** §18 lists what an
  implementation must test; the operator and both clients test
  themselves against their own understanding, which §19 explicitly says
  is not conformance. Two divergences found by hand during the first
  deployment argue the fixtures would earn their keep, because both
  fail silently: the profile's `offers` example is an array of strings
  while both clients parse objects, and the two clients rendered the
  same derived identity as upper- and lower-case hex, which would have
  made a cross-platform restore land every row under an owner the
  device does not have.
- **The operator holds the only copy.** Sealed snapshots live on one
  block volume and nothing backs it up. Acceptable while the holders
  are testers; not acceptable for anyone else, and it is an operational
  gap rather than a profile one.
- **A round trip loses an invitation's status.** The archive format
  declares invitation status at a fixed default rather than carrying
  it, so a restore cannot bring back an accepted invitation onto a
  device that does not already hold one. Closing it is a format change.
- **No incremental upload.** The abstract contract names a scheme that
  stays verifiable against a whole-snapshot reference without leaking a
  change map to the operator as unsolved design work, and this profile
  doesn't solve it. Every snapshot is a full upload — a real cost in
  bytes, time, and battery on a large history.
- **No re-bind after rotation.** Rotation is therefore destructive
  unless a client simply refuses to offer it. A re-bind proven by
  signatures from both the old and new key is the obvious shape, but
  what it does to the "no operator-controlled reset path" guarantee
  isn't obvious — which is exactly why it's deferred rather than
  guessed at.
- **The third-party-cost disclosure pattern is still open.** The
  profile requires the consequence to be stated plainly and first; it
  doesn't claim that's a *good* pattern, only an honest one.
- The abstract contract it answers to (`UI-Backup.md`) is itself a
  merged draft (0.1, August 2026), amended alongside this profile on
  three points this profile's own implementation work exposed: an
  opacity-not-containment wording fix for group secrets in a snapshot,
  a correction stating access-key rotation as a profile decision with a
  disclosed cost rather than a guaranteed property, and a declared
  `shutdownNotice` window replacing a claim that export survives
  operator shutdown outright.

## Next steps

- [Backup](backup.md) — the abstract seat this implements.
- [Identity](identity.md) — where the BIP-39 seed this profile derives
  every key from actually lives. Recovery trustee also has a hand in
  the access key's fate (§16.2 of the abstract contract), but no seat
  page exists yet for it.
