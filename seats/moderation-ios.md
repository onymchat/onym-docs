# Moderation — iOS

The reference interface implementation, and the one with the longest
running history: Apple DeviceCheck holds the two bits, an `apple/`
backend is the only writer, and an `OnymModeration` client package
validates every verdict before it ever touches a screen.

**Profile:** [`moderation/Moderation-DeviceCheck.md`](https://github.com/onymchat/onym-system/blob/main/moderation/Moderation-DeviceCheck.md)
(implements the [abstract moderation contract](moderation.md))
**Code:** [`onym-moderation/apple`](https://github.com/onymchat/onym-moderation/tree/main/apple) (Rust backend)
· [`onym-ios`](https://github.com/onymchat/onym-ios) —
`Packages/OnymModeration` (domain) and `Packages/OnymModerationUI`
(SwiftUI flows)
**Live:** `moderation.onym.app`

## Two bits, one writer

Apple stores exactly two bits per device, per developer account, and
the `apple/` backend is the only process with a write path to them:

| Bit | Mark | Set by | Cleared by |
|---|---|---|---|
| `bit0` | `case-open` | a valid interim `open-case` verdict | dismissal, superseding ban, decision-deadline default |
| `bit1` | `banned` | a valid ban verdict, at or after its `executeAfter` | expiry, reversal, new-holder appeal |

Everything else — which verdict, which case, until when — lives in the
backend's own store; the bits are a cache of its conclusions. The
backend decides nothing itself: it receives signed verdicts, checks
their **shape** mechanically, and executes the ones that conform. A
verdict missing its reasoning, naming a class outside the mandate, or
carrying an expiry longer than the consented term is refused no matter
who signed it.

## The client's side of the seam

`Packages/OnymModeration` is where the contract's client obligations
live, split by concern:

- **`ModerationMandate`** — the signed, countersigned consent artifact
  (Moderation.md §5.3), immutable once signed.
- **`DeviceAttestationProvider`** — wraps Apple `DeviceCheck`; reports
  unsupported on simulator/enterprise builds and throws
  `attestationUnavailable` rather than guessing.
- **`EnforcementBackendClient`** — talks to `apple/`; a
  `StubEnforcementBackendClient` exists for developing without a live
  backend, and it is only permitted to answer `clear` to a nil token,
  never a real implementation.
- **`GateCheckRepository`** — the client-side cadence policy: a
  default P1D check interval with a P3D offline grace period before
  the client itself refuses to treat a device as clear.
- **`VerdictValidator`** — pure, stateless shape validation against
  Moderation.md §5.6. The client never re-decides a case; it only
  checks that a signed verdict is internally consistent before
  rendering its effect.
- **`ModerationSigner`** — a seam the app target adapts to its own
  `IdentityRepository`, so private keys never cross the package
  boundary into moderation code.
- **`DeviceRecovery`** — the moderator-issued recovery-grant flow
  (REFERENCE-AUTHORITY-POLICY §6): redeeming a grant that moves a
  cleared case's record to a new enrollment. iOS has this built end to
  end, matching `apple/`'s `POST /v1/recover`.
- **`AuthorityManifestFetcher` / `AuthorityManifestValidator`** — fetch
  and pin an authority's published terms before a mandate is signed
  against them.

`Packages/OnymModerationUI` carries the SwiftUI flows a host app wires
up directly: consent (`ModerationConsentFlow`), reporting
(`ModerationReportFlow`), the gate itself (`ModerationGateFlow`,
`GateCheckRequiredView`), the banned and open-case states
(`BannedView`, `OpenCaseBanner`), device recovery
(`DeviceRecoveryFlow`), and settings
(`ModerationSettingsFlow`). The app-level adapter
(`Sources/OnymIOS/IdentityModerationSigner.swift`) is the only file
that connects `ModerationSigner`'s seam to real identity keys.

## The recovery endpoint, and what it actually proves

`POST /v1/recover` clears a device's bits for a new holder on a
**moderator's signed authorization** — nothing more. It does not, and
cannot, prove the presented device is the one the case marked:
DeviceCheck tokens are unlinkable, so no case-to-device cryptographic
binding exists to check. The interface only knows that Apple confirms
*some* banned device is presented in a session signed by the grant's
grantee. The moderator's verification of the new-holder claim is the
control — not a cryptographic link, because none exists. The grant
itself is bound to the grantee identity, single-use, carries a domain
tag distinguishing it from a verdict signed by the same key, and lapses
after 30 days. Marks still move only on verdicts: the grant re-binds
the case's record to the new enrollment, and the case's already-filed
reversal verdict is what performs the clear.

## Cross-implementation agreement, not shared code

The two sides of this binding — Swift client, Rust backend — share no
library and agree by bytes. `apple/src/payload.rs` carries fixtures
produced by running the **iOS client's own `SignedSessionPayload`**,
not a retyping of it, so a format drift fails a test instead of
silently rejecting every real signature. Canonical signing bytes
(`apple/src/canonical.rs`) remove signature fields structurally on both
sides, never by string surgery — a technique the "Honest limits"
section below flags as a real trap for any third implementation.

## Countersigning keys, and rotating one

The backend countersigns a mandate to say it witnessed *this user*
consenting to *that authority*; `MODERATION_INTERFACE_SIGNING_SEED` is
the root of those signatures. Keys are derived **per authority**
(`epoch 0` = the root seed; `epoch n` = a SHA-256 of the root, the
authority's component id, and `n`) so that rotating one relationship
never touches another — the alternative, one key for every authority,
made rotation all-or-nothing. Rotation is a four-step, gapless
handshake between this backend and each authority operator: derive the
next epoch without deploying it, the authority adds it alongside the
current key, the backend deploys and switches to signing with it, and
only then does the authority drop the old key. That last step is the
only irreversible one — it's what actually burns a compromised key.

## Honest status

- **`MODERATION_ENFORCE_SIGNATURES` defaults to `false`** on the
  backend, so verdicts with unverifiable authority signatures are
  accepted — a pre-launch default for a world where no authority
  publishes a signing key yet. The iOS client, by contrast, enforces
  both manifest and verdict signatures unconditionally. The reference
  deployment sets the backend flag `true`; any other deployment must
  set it explicitly.
- **DeviceCheck tokens are ephemeral and unlinkable by design.** The
  backend cannot tell one device from another across sessions; it can
  only confirm a real device was present at enrollment. When a banned
  identity presents a device whose bits are clean, the identity is
  refused but that device's bits are left alone — branding it would
  mark hardware the verdict never named, quite possibly a new owner's.
- **New-holder claims can't be cryptographically authenticated** — see
  [the recovery endpoint](#the-recovery-endpoint-and-what-it-actually-proves)
  above. The moderator's judgment is the whole control.
- **Canonical JSON agreement is by construction, not by spec.** Beware
  Foundation's `JSONSerialization`, which sorts keys
  case-insensitively and produces bytes the backend can't reproduce —
  a client built against a different JSON library must sort by UTF-8
  byte order to interoperate.
- **The write log is a paper control until audited.** Every mark write
  is appended to a hash-chained log naming the verdict that authorized
  it, but that log — plus an audit-seat attestation — is this
  profile's substitute for a platform-level proof of faithful writing
  (spec §8 gap 3), and no auditor has attested a deployment yet.

## Next steps

- [Moderation](moderation.md) — the technology-free contract this
  implementation answers to.
- [Moderation — Android](moderation-android.md) — the Play Integrity
  sibling, and where it deliberately diverges (no recovery endpoint
  yet).
- [Run your own authority](run-your-own-authority.md) — the judgment
  service this backend receives verdicts from.
- [Deployment](../deployment.md) — how the reference deployment brings
  the interface backends up.
