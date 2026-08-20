# Run your own authority

This application runbook takes you from a checked-out repository and a
TLS-ready host to a configured moderation authority: keys generated,
terms published, the service running, and a moderator able to decide
cases. Host provisioning, reverse-proxy configuration, process
supervision, firewalling, and operating-system hardening are outside its
scope; use the [reference deployment](../deployment.md) as the concrete
Docker Compose example. It assumes you've read the
[Moderation overview](moderation.md) — you should know what a mandate,
a manifest, and a verdict are before operating a service that issues
them.

## What you're signing up for

An authority's entire power comes from its published manifest. Users
consent to those exact bytes, and everything you do afterwards is
measured against them. In practice that means a few standing
obligations:

- **Decide on time.** Every case carries a decision deadline from your
  manifest. If you miss it, the case dismisses itself automatically —
  the software enforces this, you can't opt out. An absent moderator
  doesn't hurt the accused; it just makes your authority useless.
- **A human must be reachable.** Deciding a case requires a human's
  token (unless you explicitly configure autonomous triage, and even
  then your manifest must declare the model). The service refuses to
  start if nobody could possibly decide a case.
- **Keep your terms stable.** Once anyone has consented to your
  manifest, those bytes are frozen. Changing terms means publishing a
  *new* manifest and collecting fresh consent — never editing the old
  one.
- **Guard the signing key.** It's the root of everything you sign, and
  there is deliberately no rotation story (see
  [the one key](#the-one-key-you-must-never-lose-or-rotate) below).
- **Honor your retention promises.** Your manifest declares how long
  you keep evidence and records. A period you publish but don't enforce
  is worse than publishing none.

If that sounds acceptable, let's build one.

## Prerequisites

- **Rust** (stable; the Docker image pins `rust:1-bookworm`). The
  service is a single binary with SQLite compiled in — no database
  server, no OpenSSL.
- **A domain with TLS.** The reference deployment puts Caddy in front
  for automatic certificates. Everything below assumes
  `https://authority.example.org`.
- **The repo:**

```sh
git clone https://github.com/onymchat/onym-moderation
cd onym-moderation/authority
cargo test          # ~150 integration tests; make sure you start green
```

## Step 1 — Generate your identity

Your authority's identity is an Ed25519 key derived from a 32-byte
seed. Generate the seed and derive the public key:

```sh
export AUTHORITY_SIGNING_SEED=$(openssl rand -hex 32)
cargo run -- derive-operator-key
# → onym:key:3f8a…   (64 hex chars after the prefix)
```

Put the seed in a real secret store **now**. The printed `onym:key:…`
value is public — it goes into your manifest as `operator`, and clients
will verify every verdict you ever sign against it.

### The one key you must never lose or rotate

The seed signs your manifest, every verdict, and every recovery grant.
There is intentionally no key-rotation protocol: if you rotate the seed
while any ban is in force, every verdict you already issued stops
verifying — which, from the outside, is indistinguishable from forgery.
Rotation means publishing a new manifest with a new `operator` key and
collecting fresh mandates. So: dedicated secret, separate from your
operational credentials (admin tokens and the like), backed up, never
reused.

If the key is ever compromised, the contract requires you to report it
immediately to the interfaces that designate you.

## Step 2 — Write your manifest

The manifest is the hard part — not technically, but because it's a
commitment. Start from
[`manifest.example.json`](https://github.com/onymchat/onym-moderation/blob/main/authority/manifest.example.json)
and work through it field by field.

The skeleton:

```json
{
  "version": 1,
  "componentId": "onym:component:your-authority",
  "seat": "moderation",
  "operator": "onym:key:<the key from step 1>",
  "moderationProfileId": "onym:moderation-profile:consent-bound-v1",
  "violationClasses": [ "…" ],
  "evidenceRules": "https://authority.example.org/policy/evidence-rules",
  "reputationPolicy": "https://authority.example.org/policy/reputation",
  "newHolderAppeal": "https://authority.example.org/policy/new-holder",
  "confidentiality": "https://authority.example.org/policy/confidentiality",
  "retention": { "…": "…" },
  "statistics": "https://authority.example.org/policy/transparency",
  "validUntil": "2027-06-30T23:59:59Z"
}
```

### Violation classes: the five terms

Each class you're willing to judge declares **five terms**, and a class
missing any of them is invalid — the server refuses to boot:

```json
{
  "classId": "unsolicited-pornography",
  "definition": "https://authority.example.org/policy/unsolicited-pornography",
  "responseWindow": "P7D",
  "decisionDeadline": "P14D",
  "banTerm": "P90D",
  "appealWindow": "P30D",
  "appealEffect": "suspensive"
}
```

- `responseWindow` — how long the accused has to answer after notice.
  A ban is refused until this has fully elapsed.
- `decisionDeadline` — how long you have to decide, measured from case
  opening. Miss it and the case dismisses by default.
- `banTerm` — a duration, or the literal string `"permanent"`.
- `appealWindow` — how long after a ban an appeal can be filed.
- `appealEffect` — `"suspensive"` (the ban doesn't execute until the
  appeal window closes) or `"non-suspensive"` (it executes immediately;
  an appeal can only reverse it).

All durations are whole days in ISO 8601 form: `P7D`, `P30D`, `P365D`.
Hours (`PT12H`), zero days, and prose won't parse.

Two rules with teeth:

- **A `permanent` ban term requires an external appellate.** The
  contract demands that a permanent ban stay appealable *somewhere
  else* for as long as it's in force — `"appellate":
  "onym:component:<some-other-authority>"`. If you can't name one,
  don't declare permanent terms; use a long finite `banTerm` instead.
  (This is why the reference deployment's own manifest caps everything
  at `P365D`.)
- **Give yourself deciding room.** `decisionDeadline` must comfortably
  exceed `responseWindow` — you can't ban before the window closes, so
  the gap between them is your actual time to judge.

### Retention: promises with anchors

```json
"retention": {
  "policy": "https://authority.example.org/policy/retention",
  "unreferencedUpload": "P1D",
  "caseMedia": "P30D",
  "caseRecord": "P400D",
  "auditRecord": "P400D",
  "sanctionRecord": "P400D"
}
```

Each period is a *tail* measured from when the governed thing finishes
(a case's deadlines passing, a mark expiring) — not from arrival. The
server validates the schedule at boot: `sanctionRecord` must be at
least as long as `caseRecord` and `auditRecord`, or a still-sanctioned
case would outlive its own record.

> **A serious legal warning about `preservation`.** The example
> manifest contains a commented-out `preservation` block for classes
> that carry statutory reporting duties (such as CSAM). Declaring
> preservation for a class means your authority will **accept and take
> custody of that class's image evidence** and owes a referral to real
> authorities. An operator who is not a service provider with the
> legal protections and reporting relationship that role carries may
> commit an offence *merely by holding such material*. Without the
> block, the class refuses media evidence — which is the safe default.
> Do not enable it without legal advice.

### Publish the documents you link to

Every URL in your manifest (`definition`, `evidenceRules`,
`newHolderAppeal`, …) must actually resolve, because those documents
are what users consent to. The reference keeps them as Markdown files
in `authority/published/`, served verbatim by Caddy under
`https://<host>/policy/` as `text/markdown` — verbatim rather than
rendered, because these are the bytes that were signed off. No
`#fragment` links. See
[`PUBLISHING.md`](https://github.com/onymchat/onym-moderation/blob/main/authority/PUBLISHING.md)
for the layout, and
[`REFERENCE-AUTHORITY-POLICY.md`](https://github.com/onymchat/onym-moderation/blob/main/REFERENCE-AUTHORITY-POLICY.md)
for prose you can adapt.

### Once published, the bytes are frozen

Every mandate pins `sha256(<exact manifest bytes>)`. If you edit the
published file after anyone has consented — even to reformat whitespace
— new registrations will still work (they pin the new hash), but you've
orphaned everyone on the old terms unless the server kept its snapshot.
It does keep snapshots of every manifest it has served, and it judges
each case under the snapshot the accused's mandate pinned. But the
rule stands: **to change terms, publish a new manifest and collect
fresh consent. Never edit in place.**

## Step 3 — Configure and start

The service is configured entirely through environment variables.

### Required

| Variable | What it is |
|---|---|
| `AUTHORITY_SIGNING_SEED` | The 64-hex-char seed from step 1. Must match the manifest's `operator`, or the server exits with both values printed. |
| `AUTHORITY_MANIFEST_PATH` | Path to your manifest JSON. Read at boot, validated (class terms, retention schedule, `validUntil`), and served byte-for-byte at `/manifest.json`. |

And at least one way to decide: set `AUTHORITY_MODERATOR_TOKEN` or
`AUTHORITY_ADMIN_TOKEN` (below). If neither is set and triage isn't
autonomous, the server refuses to start — an authority nobody can
decide for would dismiss every case by default.

### Moderation access

| Variable | What it is |
|---|---|
| `AUTHORITY_MODERATOR_TOKEN` | Bearer token for the decide API, the requeue API, and the detailed health view. `openssl rand -hex 32`. |
| `AUTHORITY_ADMIN_TOKEN` | Login token for the `/admin` web panel (8-hour sessions, HttpOnly cookie). |

### Connecting to an interface

| Variable | What it is |
|---|---|
| `AUTHORITY_INTERFACE_URL` | Base URL of the interface's enforcement backend, e.g. `https://moderation.example.app`. Verdicts are pushed to `{url}/v1/verdicts`. **Unset means verdicts are signed but never delivered — no mark will ever move.** |
| `AUTHORITY_INTERFACE_TOKEN` | Bearer token for those deliveries; must match the interface's `MODERATION_AUTHORITY_TOKEN`. |
| `AUTHORITY_INTERFACE_KEY` | The interface's countersigning key(s), comma-separated `onym:key:…` values. **Empty means every mandate registration is refused** — the service looks healthy but acquires no jurisdiction. Read it from the interface's `/health` (the interface derives a distinct key per authority; use `rotatedInterfaceKeys` for your component id if present, else `interfaceKey`). Listing two keys lets the interface rotate with no gap. |

Because the interface's per-authority key doesn't exist until the
interface has booted with your authority configured, first deploys are
a two-pass dance: bring both services up with `AUTHORITY_INTERFACE_KEY`
empty, read the key from the interface's `/health`, set it, redeploy.
Until pass two, mandate registration is deliberately closed — that's a
bootstrap state, not a failure.

### Everything else

| Variable | Default | What it is |
|---|---|---|
| `AUTHORITY_BIND` | `0.0.0.0:8080` | Listen address. |
| `AUTHORITY_STORE_PATH` | `/data/authority.sqlite` | The SQLite store (WAL mode). This file *is* your authority — see [Backups](#backups). |
| `AUTHORITY_DEADLINE_SWEEP_SECS` | `300` | How often the background sweep runs: deadline defaults, delivery retries, retention deletion. |
| `AUTHORITY_QA_ALLOW_EARLY_BAN` | `false` | Lets a *human* ban before the response window closes. For test rigs only — never set this on a public authority. |
| `RUST_LOG` | `info` | Standard tracing filter. |

### First boot

```sh
export AUTHORITY_SIGNING_SEED=<from your secret store>
AUTHORITY_MANIFEST_PATH=/etc/onym/manifest.json \
AUTHORITY_STORE_PATH=/var/lib/onym/authority.sqlite \
AUTHORITY_MODERATOR_TOKEN=$(openssl rand -hex 32) \
AUTHORITY_ADMIN_TOKEN=$(openssl rand -hex 32) \
cargo run --release
```

The server validates everything up front and exits loudly (with a
usage text) on any misconfiguration: bad seed, unparseable class
terms, an inverted retention schedule, an `operator` that doesn't match
the seed, an expired `validUntil`. A clean start logs
`authority starting` with your component id, signing key, and manifest
hash, then `listening`.

Verify:

```sh
curl -s localhost:8080/health
```

```json
{
  "status": "ok",
  "authority": "onym:component:your-authority",
  "signingKey": "onym:key:…",
  "manifestHash": "…",
  "interfaceConfigured": true,
  "canDecide": true,
  "undeliverableVerdicts": 0,
  "undeliverable": null
}
```

Three things to check: `interfaceConfigured` is `true`, `canDecide` is
`true`, and `undeliverableVerdicts` is `0`. That's also your ongoing
monitoring checklist.

## Step 4 — Sign the manifest

Clients that enforce manifest signatures fetch
`/manifest.json.sig` — a detached Ed25519 signature over the exact
published bytes — and verify it against your pinned operator key. The
binary produces it:

```sh
onym-moderation-authority sign-manifest /etc/onym/manifest.json > manifest.json.sig
```

Publish the `.sig` next to the manifest. If you ever republish the
manifest, re-sign it in the same deploy step, and update the files in
an order that never leaves a signature covering different bytes
(retire the old `.sig`, move the manifest, write the new `.sig` last).

## Day-to-day moderating

### The moderator panel

`https://<your-host>/admin`, log in with `AUTHORITY_ADMIN_TOKEN`. You
get two queues — **awaiting decision** (open cases, soonest deadline
first, with urgency markers inside two days and a note on whether the
response window has closed) and **appeals awaiting review** (ordinary
appeals and new-holder claims, labeled) — plus an audit log and the
device-recovery queue. Decisions from the panel are recorded as human
decisions.

### Deciding from the command line

The decide API takes the same three dispositions the panel offers:

```sh
# Dismiss
curl -X POST https://<host>/v1/cases/$CASE_ID/decide \
  -H "Authorization: Bearer $AUTHORITY_MODERATOR_TOKEN" \
  -H 'content-type: application/json' \
  -d '{"disposition":"dismiss","reasoning":"sha256:<content address of findings>"}'
```

```sh
# Ban — the three recourse fields are mandatory
curl -X POST https://<host>/v1/cases/$CASE_ID/decide \
  -H "Authorization: Bearer $AUTHORITY_MODERATOR_TOKEN" \
  -H 'content-type: application/json' \
  -d '{
    "disposition": "ban",
    "reasoning": "sha256:<content address of findings>",
    "appealUrl": "https://authority.example.org/appeal",
    "newHolderUrl": "https://authority.example.org/new-holder",
    "authorityContact": "appeals@example.org"
  }'
```

Reasoning is required on *every* disposition, dismissals included, and
should be a content address of your written findings rather than
inline prose. A ban is refused (409/410) if any notice hasn't reached
the interface yet, if the response window hasn't elapsed, or if the
decision deadline has already passed.

To answer an appeal, add `"reviewed": "appeal"` (or `"new-holder"`)
with `"disposition": "reverse"` to overturn — a reversal goes out on
the wire as a fresh dismissal verdict that clears both marks. Include
the `claimRevision` you saw in the case status; the commit is refused
if the claim changed under you.

### When delivery gets stuck

Delivery is push-based and at-least-once: temporary failures
(interface down, 5xx, auth errors) retry forever on the sweep
interval. But if the interface *structurally refuses* a verdict three
times — `bad_request`, `verdict_invalid`, `class_outside_mandate` —
the authority stops retrying and parks it as **undeliverable**. That's
a mark that should have moved and didn't, and it blocks any ban on
that case, so it's the number to watch in `/health`.

With the moderator token, `/health` lists each stuck verdict with the
interface's own error message. Fix the underlying disagreement, then:

```sh
curl -X POST https://<host>/v1/verdicts/$VERDICT_REF/requeue \
  -H "Authorization: Bearer $AUTHORITY_MODERATOR_TOKEN"
```

## Optional: model-assisted triage

The authority can run a local safety classifier over incoming cases.
Three modes via `AUTHORITY_TRIAGE_MODE`:

- `off` (default) — humans see raw cases.
- `advisory` — the model's assessment appears alongside the case in the
  panel; a human still decides.
- `autonomous` — the model's recommendation is applied as the decision.
  Your manifest must then declare the exact model in `modelProfile`
  (id and digest), because users are consenting to be judged by it.
  The server refuses to boot on any mismatch.

Pick a built-in profile (`AUTHORITY_TRIAGE_PROFILE`, e.g.
`gpt-oss-safeguard-20b`) or supply your own
(`AUTHORITY_TRIAGE_PROFILE_PATH`), and point `AUTHORITY_TRIAGE_URL` at
an OpenAI-compatible endpoint. One deliberate constraint: **the model
must run on the same host**. Case evidence was disclosed to *you* for
adjudication; sending it to a third-party API is a further disclosure,
so the server exits if the triage URL resolves off-host. There are
also deliberately no threshold or prompt overrides — those live in the
profile your manifest pins.

## Operations

### Backups

Back up the directory holding `AUTHORITY_STORE_PATH`. That one SQLite
file holds your cases, reports, reporter track records, every verdict
you've signed, evidence blobs, and — crucially — the snapshot of every
manifest version anyone ever consented to. Losing it loses the terms
your live cases must be judged under, and it is also the accused's
proof of how their case ended. The manifest file itself lives outside
the store, but its snapshots don't.

### Monitoring

- `GET /health` is unauthenticated — wire it into your uptime checks.
  Alert on `undeliverableVerdicts > 0`.
- Logs are structured `tracing` output. The lines worth alerting on:
  `interface permanently refused a verdict` and
  `decision deadline passed; case dismissed by default` (each one is a
  case you failed to decide in time).
- The deadline sweep uses wall-clock time, so downtime across a
  deadline is honored on the way back up — a case whose deadline
  passed while you were down dismisses at the next sweep.

### `validUntil` expiry

An expired `validUntil` doesn't stop the service: existing cases run
to completion, but every new mandate and new case is refused, and the
log complains loudly. Renew by publishing a successor manifest before
the date passes.

## Things that will bite you

A short list of misconfigurations that produce a *healthy-looking*
authority that quietly does nothing, plus the rules with no undo:

| If you… | Then… |
|---|---|
| leave `AUTHORITY_INTERFACE_KEY` empty | every mandate registration is refused; you never gain jurisdiction over anyone |
| leave `AUTHORITY_INTERFACE_URL` empty | verdicts are signed and stored but no device mark ever moves |
| edit a published manifest in place | consent breaks silently; existing mandates no longer match the served bytes |
| rotate `AUTHORITY_SIGNING_SEED` | every verdict already issued stops verifying — indistinguishable from forgery downstream |
| let `validUntil` lapse | new mandates and cases are refused until you publish a successor |
| point triage at a wrong/dead URL | no assessment ever lands; in autonomous mode every case runs to its deadline and dismisses |
| ignore `undeliverableVerdicts` | notices aren't reaching the interface, and every ban on those cases is refused |

And a client-side trap worth telling your integrators about: signing
bytes must be canonicalized with keys sorted by **UTF-8 byte order**.
Foundation's `JSONSerialization` sorts case-insensitively and produces
bytes the authority can't reproduce — every signature from such a
client fails.

## Reference deployment

The Onym deployment in
[`onym-infra`](https://github.com/onymchat/onym-infra) runs this exact
service behind Caddy on one droplet, including the two-pass interface
bootstrap and the manifest sign-and-publish ordering — see
[Deployment](../deployment.md). The repo's own
[`SKILL.md`](https://github.com/onymchat/onym-moderation/blob/main/authority/SKILL.md)
is the condensed runbook, including its "refuse to deploy if"
checklist.
