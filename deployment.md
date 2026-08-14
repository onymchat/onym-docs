# Deployment

[`onym-infra`](https://github.com/onymchat/onym-infra) brings every
server-side seat up on one DigitalOcean droplet via Docker Compose.

| Service | Host | Seat |
|---|---|---|
| Caddy | — | reverse proxy, automatic TLS |
| strfry | `nostr.onym.app` | [courier — message](seats/courier.md) |
| blossom | `blossom.onym.app` | [courier — blob](seats/courier.md) |
| relayer | `relayer.onym.app` | [notary](seats/notary.md) |
| moderation | `moderation.onym.app` | [moderation](seats/moderation.md) — enforcement |
| authority | `authority.onym.app` | [moderation](seats/moderation.md) — judgment |

One box is a cost decision and nothing depends on it. The authority
delivers verdicts to the interface's **public** hostname rather than over
the private network, so the day the interface moves to another operator
that address points elsewhere and nothing else changes — and this
deployment exercises the same TLS + token + signature path everyone else
must use.

The one thing that genuinely must stay private is the triage model
container, if enabled: case evidence was disclosed for adjudication, and
sending it to a third party is a further disclosure.

## First run

```sh
git clone --recurse-submodules <repo> onym-infra && cd onym-infra
cp .env.example .env                     # DO_API_KEY, CF_API_TOKEN, hosts, size
cp relayer.env.example relayer.env       # RELAYER_SECRET_KEY (required)
cp moderation.env.example moderation.env # DeviceCheck key + ids, interface seed
cp authority.env.example authority.env   # signing seed + admin token (required)
./deploy/digitalocean/deploy.sh
```

The script creates or adopts an `s-1vcpu-2gb` droplet by name, adds a 2 GB
swapfile, upserts **DNS-only** Cloudflare A records, syncs and brings the
stack up. Re-runs update the box.

Two traps:

- The swapfile is written by cloud-init, which runs only at droplet
  **creation**. Three Rust builds share 2 GB; adding swap later is a manual
  `ssh` job.
- The Cloudflare records must stay grey-cloud. Proxying breaks Caddy's ACME
  challenge and the Nostr `wss://` connection.

## Bootstrapping the moderation pair

Only the interface countersigning key settles after a boot, so the first
deploy is two passes with no unsigned enforcement window:

1. Generate both seeds, leave `AUTHORITY_INTERFACE_KEY` empty, keep
   `MODERATION_ENFORCE_SIGNATURES=true`, deploy. The script derives the
   authority operator key before startup and materializes a matching
   manifest, so every accepted verdict is signed from the first request.
2. Read the interface's public key and re-run:

```sh
curl -fsS https://moderation.onym.app/health \
  | python3 -c 'import json,sys; print(json.load(sys.stdin)["interfaceKey"])'
```

Until pass two the authority refuses newly registered mandates. That is a
closed bootstrap state, not permission to accept unsigned verdicts.

## Manifest and policy documents

The authority's manifest is materialized at deploy time from the template
in the `moderation` submodule, with this deployment's operator key and
`AUTHORITY_HOST`. Changing any of those bytes after consent creates a new
manifest hash — existing mandates stay bound to the old terms and fresh
consent is required.

Caddy serves the ten policy documents from
`moderation/authority/published/` as `text/markdown; charset=utf-8`, one
per term and per violation class, with no `#fragment` links. They are
served verbatim rather than rendered: these are the bytes that were signed
off.

`manifest.json.sig` is the detached Ed25519 signature over the exact
published bytes — **base64 of 64 raw bytes plus a trailing LF, 89 bytes
total**. Consumers must trim whitespace before decoding. `deploy.sh` signs
inside the authority image, retires the old signature, moves the new
manifest into place, then writes the new signature last, so the published
pair can never cover different bytes. A signing failure leaves the
previous pair intact; a later interruption degrades to 404 → soft-verify.

Order matters against the client: this asset must be live and verified
before `ModerationTrust.enforceManifestSignatures` is flipped on in
`onym-ios`, since under enforcement a 404 rejects the manifest and blocks
consent outright.

## Rotating an authority's countersigning key

The interface derives a separate key per authority from
`MODERATION_INTERFACE_SIGNING_SEED` plus a per-authority epoch in
`MODERATION_INTERFACE_KEY_EPOCHS`. Empty means epoch 0 — the seed used
directly, the un-rotated state.

This is about rotation, not containment: all derived keys live in the same
process as the root. What it buys is burning one relationship without
invalidating every countersignature ever issued to everyone.

`AUTHORITY_INTERFACE_KEY` takes a comma-separated list, which is what makes
the cutover gapless:

1. Derive the next key without deploying it (bump the epoch in a scratch
   environment, read `rotatedInterfaceKeys` from `/health`).
2. **Add** it beside the current one, deploy. Both verify; nothing changes.
3. Set the authority's entry in `MODERATION_INTERFACE_KEY_EPOCHS`, deploy.
   Earlier countersignatures still verify.
4. Remove the old key, deploy.

Step 4 is the only irreversible one, and it is the point. Any other
ordering — or a single-value swap — refuses every registration for that
authority until both sides agree.

A wrong component id parses fine, silently leaves that authority on epoch
0, and surfaces as a *signature* error rather than a configuration one.
The interface names every configured id in its boot log beside the key it
produced; compare that against what the mandates carry.

## CI

`.github/workflows/deploy.yml` runs the same script from a manual
`workflow_dispatch`, writing all four env files from Secrets and Variables.
This is also the relayer's deployment path — `onym-relayer` releases now
publish manifests only.

`AUTHORITY_INTERFACE_KEY` is a Variable, not a Secret: it is a public key,
and it does not exist until the interface has booted once.

## Discovery provider (in review)

Nothing above serves `discovery.onym.app` yet — the signed
[discovery](seats/discovery.md) provider has no deployment, and the
hostname does not resolve. A manual deploy workflow is in review as
[`onym-discovery` #4](https://github.com/onymchat/onym-discovery/pull/4):
a `workflow_dispatch` `deploy.yml` that builds the reference CLI, signs
and chains the snapshot onto the previously **published** one (a
genesis publish is an explicit input, not a guess), verifies everything
exactly as a client would before a byte leaves the runner, then rsyncs
the static tree onto the **same droplet** and adds a Caddy vhost for
it. Signing seeds (`DISCOVERY_OPERATOR_SEED` and the courier/blossom
seat seeds) live as Actions secrets, with a `skip_signing` path for
operators who sign locally instead; the job runs in a `production`
environment that must be configured with required reviewers before the
first dispatch, or it gates nothing. Until that PR merges and runs,
this section describes a review branch, not the deployment.

## Operating

```sh
ssh root@<DROPLET_IP> && cd /opt/onym-infra
docker compose ps
docker compose logs -f caddy        # cert issuance / renewal
docker compose logs -f authority
```

Both moderation services answer `GET /health`; `deploy.sh` checks for 200
at the end of a run.

The human moderation queue is at `https://$AUTHORITY_HOST/admin`, behind
`AUTHORITY_ADMIN_TOKEN`. The authority refuses to start without this human
route, and deploy catches an empty token before touching the droplet.
