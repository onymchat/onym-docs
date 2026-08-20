# Moderation — Android

The Google Play Integrity device-recall profile, and the interface
seat's newest binding: a by-copy sibling of the
[iOS implementation](moderation-ios.md), agreeing with it by bytes and
tests rather than shared code, and trailing it by one endpoint on
purpose.

**Status:** Running alpha implementation; device recovery is not
implemented and durable recall depends on Google beta access.

**Profile:** [`moderation/Moderation-Device-Recall.md`](https://github.com/onymchat/onym-system/blob/main/moderation/Moderation-Device-Recall.md)
(implements the [abstract moderation contract](moderation.md))
**Code:** [`onym-moderation/android`](https://github.com/onymchat/onym-moderation/tree/main/android) (Rust backend)
· [`onym-android`](https://github.com/onymchat/onym-android) —
`modules/moderation` (domain) and `modules/moderation-ui` (Compose
flows)
**Live:** `moderation-android.onym.app`

## Three values, one writer

Play Integrity's device-recall API exposes three per-device values,
scoped to the whole Play developer account rather than to one app.
This backend is the only process holding the service-account key that
can write them:

| Recall value | Abstract mark | Notes |
|---|---|---|
| `bitFirst` | `case-open` | |
| `bitSecond` | `banned` | |
| `bitThird` | reserved | never written — `RecallChanges` has no field for it |

As with iOS, the backend decides nothing: it validates a verdict's
shape mechanically and executes the ones that conform. Writes specify
only the values a transition changes, and every write attempt —
accepted or refused — lands on the hash-chained write log with the
verdict (or clearing rule) that authorized it.

## A session, end to end

1. The app fetches a single-use challenge from `POST /v1/challenge`,
   builds the length-prefixed signed session payload
   (`android/fixtures/README.md` is the normative wire format), signs
   it with the identity key, and passes
   `base64url-nopad(SHA-256(payload))` to Play Integrity's
   `setRequestHash`.
2. The backend verifies the identity signature over the recomputed
   payload, claims the session signature and the challenge (both
   single-use, bounded skew / TTL), then decodes the integrity token
   through Google.
3. A **five-condition classifier** (spec §5.2) runs before any recall
   value is read: a fresh, matching `requestDetails` including
   `requestHash`; `PLAY_RECOGNIZED` against an expected signing-cert
   digest; `LICENSED`; `MEETS_DEVICE_INTEGRITY`; and the `deviceRecall`
   object actually present. Any single failure answers
   `checkRequired` — never a clean result by default.
4. Reconciliation folds the stored verdicts into intended marks, writes
   only the values that changed (Google documents up to 30 seconds of
   read lag, so rewrites inside a short propagation grace window are
   skipped), and answers.

An executed ban meeting clean values on the device is read as a
*different device* presenting the same identity: the identity is
refused, but the device is never branded — integrity tokens are
request artifacts, not device identifiers, so that's the only safe
reading.

## What the client had to build without a client

The backend's fixtures were generated from `src/payload.rs`
first — "no Android client existed when this profile was
implemented," in the backend README's own words. `modules/moderation`
in `onym-android` (`SignedSessionPayload.kt` and the rest) is the
by-copy Kotlin port that reproduces those fixtures byte-for-byte, the
same relationship iOS has with `apple/`, just built in the reverse
order. `CaseAppeal.kt` documents itself explicitly as "the Android port
of iOS's `AppealSubmission`" — the whole module set
(`AuthorityClient`, `ModerationMandate`, `GateCheckRepository`,
`VerdictValidator`-equivalent logic, `PlayIntegrityAttestationProvider`
wrapping `IntegrityManagerFactory`/`StandardIntegrityManager`) mirrors
the iOS package structure closely, and `modules/moderation-ui` mirrors
`OnymModerationUI`'s flow set (consent, gate, banned state, open-case
banner, settings, case appeal).

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/health` | Liveness + configuration surface |
| `POST` | `/v1/challenge` | Single-use challenge for the signed session payload / `requestHash` |
| `POST` | `/v1/enroll` | First-session enrollment → `deviceBinding` |
| `POST` | `/v1/mandates/countersign` | Countersignature over the exact mandate bytes the user signed |
| `POST` | `/v1/gate-check` | Reconciles recall values → `clear` / `caseOpen` / `banned` / `checkRequired` |
| `POST` | `/v1/recover` | **501 — deliberately unimplemented.** See [Deferred: recovery](#deferred-recovery) below |
| `POST` | `/v1/verdicts` | Receives, validates, and executes or queues a verdict |
| `GET` | `/v1/write-log` | The hash-chained log, and whether its chain verifies |

Gate results are internally tagged
(`{"status":"clear"}`, `{"status":"caseOpen","notices":[…]}`, …) — the
Android profile's own schema, distinct from iOS's Swift-`Codable` `_0`
shape. The two backends do not need to agree on this; only the bytes
each speaks to its own client matter.

## Deferred: recovery

Device recovery — a moderator-issued grant that moves a cleared case's
record to a new holder's enrollment — exists on iOS
([`DeviceRecovery`](moderation-ios.md#the-recovery-endpoint-and-what-it-actually-proves))
and deliberately does not exist here yet. `POST /v1/recover` answers
501, and the client has no equivalent of iOS's `DeviceRecoveryFlow`
either. Refusal responses on this endpoint still carry the authority's
contact and new-holder-claim routes, so a device isn't left with a
silent dead end — but a banned Android device whose enrollment doesn't
survive a reinstall currently has **no machine path back**. Building
it means re-adding the `apple/` backend's `binding_for_ingest`
verdict-routing logic here as well, not just the endpoint.

## Honest status

- **The recovery gap above** is the largest asymmetry with iOS, on
  both the backend and the client.
- **Device recall access is gated on Google.** Until Google grants
  device-recall beta access for the linked Play Console account, no
  integrity token carries a `deviceRecall` object at all, so a strict
  gate answers `checkRequired` to every device. The interim escape
  hatch, `MODERATION_REQUIRE_RECALL=false`, lets gating proceed on
  prerequisites 1–4 alone when the object is absent — but this is
  disclosed plainly in the backend's own docs as turning **device-level
  ban persistence off, not merely degraded**: no recall writes happen,
  a wipe or reinstall sheds a device's marks, and enforcement rests on
  identity-level refusal alone. Flip it back to `true` the day the
  grant lands; `/health`'s `requireRecall` field confirms the redeploy
  took.
- **Device recall values are Play-developer-account-wide** (spec §1,
  §8 gap 2) — every app in the account reads and can write the same
  three values, so the account must be dedicated to interfaces that
  share this exact bit contract. An app transfer abandons the marks.
- **Retention is a three-year lease**, refreshed by reads and writes
  (spec §8 gap 3) — a device absent longer than that can return clean.
  Identity-level refusal is unaffected and persists regardless.
- **A present-but-empty recall object with every prerequisite passing**
  is the profile's clean, never-written state — and also its
  irreducible ambiguity (spec §8 gap 6): the backend cannot distinguish
  "never marked" from certain edge cases, and monitors the situation
  via a `recall_empty_result` tracing event rather than claiming
  certainty it doesn't have.
- **The exact `deviceRecall:write` request field names have no Google
  sandbox to test against** — they're re-verified against the live API
  on first deploy, flagged with a `NOTE` in `src/play_integrity.rs`.
- **`MODERATION_ENFORCE_SIGNATURES` defaults to `false`**, the same
  pre-launch posture as `apple/` — set it `true` before any real
  authority is configured against a production deployment.

## Next steps

- [Moderation](moderation.md) — the technology-free contract this
  implementation answers to.
- [Moderation — iOS](moderation-ios.md) — the Apple DeviceCheck
  sibling this profile was built by-copy against, including the
  recovery flow Android still lacks.
- [Run your own authority](run-your-own-authority.md) — the judgment
  service this backend receives verdicts from.
- [Deployment](../deployment.md) — how the reference deployment brings
  the interface backends up.
