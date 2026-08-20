# Moderation

Moderation in Onym works differently from what you may be used to. There
is no platform trust-and-safety team with power over everyone. Instead,
**anyone can run a moderation authority**, and an authority only has
power over people who explicitly agreed to its published terms — before
any dispute existed.

Here's the whole idea in one paragraph: when a user joins through an
interface (an app), they sign a **mandate** — a small consent document
that names one authority and pins the exact terms it published at that
moment. Later, if someone who received abusive content reports it, the
authority opens a **case**, notifies the accused, waits out a response
window the accused agreed to, and then a human moderator decides. The
decision is a signed **verdict**. The authority itself cannot punish
anyone — it hands the verdict to the interface, which executes it as a
durable **mark** on the offending device. If the authority never
decides, the case is dismissed automatically. Silence can never become
a punishment.

**Contract:** [`moderation/Moderation.md`](https://github.com/onymchat/onym-system/blob/main/moderation/Moderation.md)
· profiles: [DeviceCheck](https://github.com/onymchat/onym-system/blob/main/moderation/Moderation-DeviceCheck.md),
[device recall](https://github.com/onymchat/onym-system/blob/main/moderation/Moderation-Device-Recall.md)
**Code:** [`onym-moderation`](https://github.com/onymchat/onym-moderation) (Rust)
· clients: [`onym-ios`](https://github.com/onymchat/onym-ios),
[`onym-android`](https://github.com/onymchat/onym-android)

Want to operate a judgment service yourself? Jump straight to
[Run your own authority](run-your-own-authority.md). Building or
auditing an interface instead? See
[Moderation — iOS](moderation-ios.md) or
[Moderation — Android](moderation-android.md) for the platform-specific
half of the contract.

## The words you'll keep seeing

| Term | What it means |
|---|---|
| **Authority** | An independently operated judgment service. It publishes terms, hears cases, and signs verdicts. Anyone may run one; authorities compete to be chosen. |
| **Interface** | The app vendor's enforcement service. It holds the platform keys (Apple DeviceCheck, Play Integrity) and is the only party that can write device marks. |
| **Manifest** | The authority's published terms: which violations it judges, how long you have to respond, how long bans last, how to appeal. Signed, and pinned byte-for-byte by every mandate. |
| **Mandate** | A user's signed, interface-countersigned consent to one authority under one exact manifest. No mandate, no jurisdiction — full stop. |
| **Case** | One open matter per accused person per violation class. Later reports about the same thing join the existing case instead of opening new ones. |
| **Verdict** | A signed, reasoned, expiring decision: open-case, dismiss, or ban. The only thing that can ever move a mark. |
| **Mark** | A per-device bit the interface writes when executing a verdict: `case-open` or `banned`. Marks survive app reinstalls because they live with the device platform, not the app. |

## Two services, two operators — on purpose

The seat is deliberately split into two services that are meant to be
run by **different organizations**:

- **`authority/`** holds the verdict signing key and the judgment. It
  can open cases, decide them, and sign verdicts. It has *no code path*
  that writes a device mark.
- **`apple/`** (the interface's Apple DeviceCheck backend) and
  **`android/`** (the interface's Google Play Integrity backend) each
  hold one platform's attestation key. Either can read and write its
  own platform's device bits — but only when executing a verdict that
  validates against the terms the user consented to. Neither can
  originate a verdict. See [Moderation — iOS](moderation-ios.md) and
  [Moderation — Android](moderation-android.md) for how each binds the
  contract to its platform, including where the Android binding still
  trails the iOS one.

None of these share a library with the authority, or with each other.
They agree on bytes over the wire, and each side pins that agreement
with its own tests. An authority that could write marks, or an
interface that could invent verdicts, would collapse the whole design —
one party would again hold both judgment and enforcement.

## How a case flows

1. **Consent, long before any trouble.** At onboarding the user signs a
   mandate naming the authority; the interface countersigns it and the
   authority stores it. This is what gives the authority jurisdiction.
2. **A report arrives.** Only someone who *received* the content can
   report it, and only by disclosing what they received along with an
   authenticity proof — a signature by the accused over that exact
   content. The authority never scans anything and cannot ask for "the
   whole conversation." Content without a proof is just a complaint.
3. **The case opens and the accused is notified.** The authority signs
   an interim `open-case` verdict. The interface sets a `case-open`
   mark and shows the accused a notice: what class, what evidence, and
   the two deadlines they agreed to — the response window and the
   decision deadline.
4. **The accused responds — or doesn't.** Responses are accepted even
   late. Answering early never shortens the window; a ban is refused
   until the full consented window has elapsed.
5. **A human decides.** A moderator reviews the case and dismisses or
   bans, always with signed reasoning. Deciding is the only path from a
   report to a sanction, and it requires a human's token — there is no
   automatic escalation. (An optional local model can *triage* cases,
   but its recommendation only becomes a decision in a mode the
   manifest itself must declare.)
6. **The verdict travels.** The authority delivers the signed verdict
   to the interface, together with the exact manifest bytes the
   accused's mandate pinned. The interface validates everything
   mechanically — signature, keys, class, deadlines — and only then
   moves the mark.
7. **Appeal, expiry, or default dismissal.** Bans carry an appeal
   window and an expiry date measured from execution; the interface
   clears the mark at expiry on the verdict's own authority. And if the
   decision deadline passes with no decision at all, the case is
   **dismissed by default** and the mark cleared. An authority that
   stalls loses the case, never the accused.

## The promises

These are the refusals the design is built around — each one closes a
familiar abuse:

- **No jurisdiction without consent.** Reports are accepted only
  against an accused whose mandate names this authority, from a
  reporter whose own mandate names it too. Anything else is refused as
  `no_jurisdiction`. A moderator imposed after the offense would be
  chosen by the accuser.
- **No evidence without authenticity.** Every disclosed item must
  verify against the accused's key. Screenshots and hearsay don't open
  cases.
- **No sanction before notice.** Every notice must have reached the
  interface and the full response window must have elapsed before a ban
  can even be entered.
- **Undecided is dismissal.** Every case carries a decision deadline
  from the manifest, and the default at that deadline is dismissal with
  the mark cleared. A stalled case can never hold a device hostage.
- **Degrade toward blocking, never toward silence.** If the interface
  loses its DeviceCheck credentials, the gate answers `checkRequired`
  for everyone rather than quietly going unmoderated.
- **Reports are free, and reporting is never paid.** No bounties, no
  per-ban revenue — an authority may only charge interfaces flat or
  per-report-adjudicated fees, so it earns nothing from opening weak
  cases or banning eagerly.

## The API at a glance

Full request shapes, authentication schemes, and error codes live in the
[authority README](https://github.com/onymchat/onym-moderation/blob/main/authority/README.md);
this is the map.

**Authority** (run by the judgment operator):

| Route | What it's for |
|---|---|
| `GET /manifest.json` | The published terms, served byte-for-byte (`.sig` alongside is a detached Ed25519 signature) |
| `POST /v1/mandates` | An interface registers a user's countersigned mandate |
| `POST /v1/reports` | A signed report with authenticity proofs |
| `PUT /v1/evidence-blobs/:sha256` | Image evidence upload (JPEG/PNG, ≤ 4 MiB) |
| `POST /v1/cases/:id/respond` | The accused's response |
| `POST /v1/cases/:id/appeal` | An appeal, or a new-device-holder claim |
| `GET /v1/cases/:id/status` | Case status, for parties only |
| `POST /v1/cases/:id/decide` | The moderator's judgment (bearer token) |
| `POST /v1/verdicts/:ref/requeue` | Retry a verdict the interface refused, after repair |
| `GET /admin` | The human moderator panel |
| `GET /health` | Signing key, manifest hash, delivery backlog |

Party credentials for status queries travel in `X-Onym-Key`,
`X-Onym-Timestamp`, `X-Onym-Signature` headers — never in the URI, so
they never land in an access log. The signature covers
`query-status:<caseId>:<timestamp>` and expires after five minutes.
Knowing a case id proves nothing; it is not a credential.

**Interface** (run by the app vendor; shown here as one shape, but
`apple/` and `android/` are separate deployments with separate keys —
see [iOS](moderation-ios.md) / [Android](moderation-android.md) for
what differs):

| Route | What it's for |
|---|---|
| `POST /v1/enroll` | First session → the device binding a mandate will carry |
| `POST /v1/mandates/countersign` | Countersigns the user's mandate |
| `POST /v1/gate-check` | Reconciles device bits → `clear` / `caseOpen` / `banned` / `checkRequired` |
| `POST /v1/verdicts` | Receives, validates, and executes or queues a verdict |
| `POST /v1/recover` | Redeems a moderator-issued recovery grant |
| `GET /v1/write-log` | Append-only, hash-chained log of every mark write |
| `GET /health` | DeviceCheck configured? enforcement on? interface public key |

Verdicts arrive wrapped in an envelope, because the interface must judge
them against the terms the user actually consented to — not whatever the
authority publishes today:

```json
{
  "verdict": { "...": "the signed verdict object" },
  "consentedManifest": "<base64 of the manifest's exact bytes>"
}
```

## Honest limits

This is alpha software and the contract is candid about the distance
between spec and code. The gaps most worth knowing:

- **The interfaces' signature enforcement defaults off.** Both
  `apple/` and `android/` ship `MODERATION_ENFORCE_SIGNATURES=false`
  (the reference deployment sets it `true`); the iOS client, by
  contrast, enforces both manifest and verdict signatures
  unconditionally. Context: the consent loop itself closed only
  recently — the iOS client now registers the finalized,
  interface-countersigned mandate with the authority from its consent
  flow — so treat the loop as freshly wired, not battle-tested.
- **Android has no device-recovery path yet.** `apple/`'s
  `POST /v1/recover` is fully built; `android/`'s answers 501 by
  design, and the Kotlin client has no equivalent flow. A banned
  Android device whose enrollment doesn't survive a reinstall
  currently has no machine path back. See
  [Moderation — Android](moderation-android.md#deferred-recovery).
- **New-holder claims can't be authenticated.** A device's new owner is,
  by definition, not the mandated identity — the claim path exists but
  is honesty-based and capped, not proof.
- **Canonical JSON is by construction, not by spec.** Both sides remove
  signature fields structurally and sort keys by UTF-8 byte order; the
  agreement is pinned by tests between these two implementations, not
  written down as a standalone spec. (Beware: Foundation's
  `JSONSerialization` sorts keys case-insensitively and will produce
  bytes the authority can't reproduce.)
- **Android device recall is gated on Google.** Play Integrity's
  device-recall beta must be granted per Play Console account before
  bans persist at the device level; until then an interim flag trades
  that persistence away rather than blocking gating entirely. See
  [Moderation — Android](moderation-android.md#honest-status).
- **External appeal routing isn't built.** A manifest can name an
  external appellate authority, but nothing routes to it yet.

## Next steps

- [Run your own authority](run-your-own-authority.md) — the step-by-step
  operator guide.
- [Moderation — iOS](moderation-ios.md) — the Apple DeviceCheck
  interface, backend and client.
- [Moderation — Android](moderation-android.md) — the Google Play
  Integrity interface, backend and client, including where it still
  trails iOS.
- [Deployment](../deployment.md) — how the Onym reference deployment
  brings these services up on one box.
