# Moderation

Consent-bound authorities that receive signed reports, run a case with
notice and a response window, and issue signed verdicts an interface
executes as durable per-device marks.

**Contract:** [`moderation/Moderation.md`](https://github.com/onymchat/onym-system/blob/main/moderation/Moderation.md)
· profiles: [DeviceCheck](https://github.com/onymchat/onym-system/blob/main/moderation/Moderation-DeviceCheck.md),
[device recall](https://github.com/onymchat/onym-system/blob/main/moderation/Moderation-Device-Recall.md)
**Code:** [`onym-moderation`](https://github.com/onymchat/onym-moderation) (Rust)

## The split

The seat is two services, meant for **two different operators**. They
share no library — they agree on bytes, and each pins that agreement
with its own tests.

| | `authority/` | `apple/` |
|---|---|---|
| Holds | verdict signing key, procedure, judgment | the Apple DeviceCheck key |
| Can | open, decide, sign | read and write device bits |
| Cannot | write a mark — no code path exists | originate a verdict |

An authority that could write marks, or an interface that could
originate verdicts, collapses the seat.

## Authority API

| | | |
|---|---|---|
| `GET` | `/manifest.json` | published terms, verbatim bytes (+ `.sig`, detached Ed25519) |
| `POST` | `/v1/mandates` | interface registers a user's mandate |
| `POST` | `/v1/reports` | signed report with authenticity proofs |
| `POST` | `/v1/cases/:id/respond` | the accused's response |
| `POST` | `/v1/cases/:id/appeal` | appeal, or new-holder claim |
| `GET` | `/v1/cases/:id/status` | party credential required |
| `POST` | `/v1/cases/:id/decide` | moderator judgment (bearer token) |
| `POST` | `/v1/verdicts/:ref/requeue` | retry a repaired delivery refusal |
| `GET` | `/admin` | moderator panel |
| `GET` | `/health` | signing key, manifest hash, delivery backlog |

`decide` is the only path from a report to a sanction and it requires a
human's token. There is no automatic escalation.

`query-status` credentials travel in `X-Onym-Key`, `X-Onym-Timestamp`,
`X-Onym-Signature`; the signature covers `query-status:<caseId>:<timestamp>`
and expires in five minutes. Never in the URI, never in an access log.
A case id is not a credential.

## Interface API

| | | |
|---|---|---|
| `POST` | `/v1/enroll` | first session → the vendor-local `deviceBinding` a mandate carries |
| `POST` | `/v1/mandates/countersign` | interface countersignature over the user's mandate |
| `POST` | `/v1/gate-check` | reconcile the bits → `clear` / `caseOpen` / `banned` / `checkRequired` |
| `POST` | `/v1/verdicts` | validate mechanically, store, execute or queue |
| `POST` | `/v1/recover` | redeem a moderator-issued recovery grant |
| `GET` | `/v1/write-log` | append-only hash-chained log, with chain verification |
| `GET` | `/health` | DeviceCheck configured? enforcement on? interface public key |

Verdicts arrive as an envelope, not a bare verdict — the manifest travels
as exact bytes because the mandate pins them:

```json
{
  "verdict": { "...": "the signed verdict object" },
  "consentedManifest": "<base64 of the manifest's exact bytes>"
}
```

## Refusals worth knowing

- **No jurisdiction without consent** — reports are accepted only against
  an accused who signed a mandate naming this authority, from a reporter
  whose own mandate names it. Otherwise `no_jurisdiction`.
- **No evidence without authenticity** — every disclosed item verifies
  against the accused's key. Content without a proof is a complaint.
- **No sanction before notice** — a ban is refused until the consented
  response window has *elapsed*. Answering early does not shorten it.
- **Degrade toward blocking** — with no DeviceCheck credentials the gate
  answers `checkRequired` for everyone, never "unmoderated".

## Run

```sh
cd authority && cargo test
export AUTHORITY_SIGNING_SEED=$(openssl rand -hex 32)
cp manifest.example.json /tmp/manifest.json
AUTHORITY_MANIFEST_PATH=/tmp/manifest.json \
AUTHORITY_STORE_PATH=/tmp/authority.sqlite cargo run
# read `signingKey` from /health, set it as `operator` in the manifest, restart
```

```sh
cd apple && cargo test
MODERATION_INTERFACE_SIGNING_SEED=$(openssl rand -hex 32) \
MODERATION_STORE_PATH=/tmp/moderation.sqlite cargo run
curl -s localhost:8080/health
```

## Gaps

- **Nothing calls `accept-mandate` yet.** The endpoint is implemented and
  tested, but `apple/` does not POST the countersigned mandate and the iOS
  client has no registration operation. Jurisdiction is seeded by hand;
  the end-to-end consent path is not closed.
- **The new-holder claim cannot be authenticated** — a new owner is by
  definition not the mandated identity.
- **Canonical JSON is unspecified.** Both sides remove signature fields
  structurally and sort keys by UTF-8 byte order; the agreement is by
  construction between these two implementations, not by spec.
