# Deployment

[`onym-infra`](https://github.com/onymchat/onym-infra) brings the
reference Courier, Stellar Notary, iOS Moderation, and Backup services
up on one DigitalOcean droplet via Docker Compose. This page covers the services
in the table below. The signed Discovery publisher is deployed by its
own workflow, as described later, and the live Android Moderation
backend's deployment is not documented in this repository.

| Service | Host | Seat |
|---|---|---|
| Caddy | — | reverse proxy, automatic TLS |
| strfry | `nostr.onym.app` | [courier — message](seats/courier-nostr.md) |
| blossom | `blossom.onym.app` | [courier — blob](seats/courier-nostr.md) |
| relayer | `relayer.onym.app` | [notary — Stellar](seats/notary-stellar.md) |
| moderation | `moderation.onym.app` | [moderation](seats/moderation.md) — enforcement |
| authority | `authority.onym.app` | [moderation](seats/moderation.md) — judgment |
| backup | `backup.onym.app` | [backup — Object-HTTP](seats/backup-object-http.md) |

One box is a cost decision and nothing depends on it. The authority
delivers verdicts to the interface's **public** hostname rather than over
the private network, so the day the interface moves to another operator
that address points elsewhere and nothing else changes — and this
deployment exercises the same TLS + token + signature path everyone else
must use.

The backup operator is the one service here that stores bytes it cannot
read, and the one whose disk is a hazard to everything else. Its sealed
snapshots sit on a **separate block volume**, not the droplet's root
filesystem: they are the only thing on this box measured in gigabytes,
and a full root disk would stop the authority recording a verdict and
the relay accepting an event. The deploy refuses to run if that volume
is not mounted and prepared, and the container refuses to start without
a sentinel file inside it — so a reboot where the mount does not return
fails loudly instead of quietly writing snapshots to the root disk.

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

The relayer's signed [notary operator manifest](seats/notary-stellar.md) ships
inside its image: `onym-relayer`'s `sign-manifest.yml` workflow signs
in CI and commits the exact bytes under `manifest-signed/`, the
Dockerfile copies them to `/srv/operator-manifest/`, and this repo's
compose file points `RELAYER_OPERATOR_MANIFEST` at that copy. After a
re-sign, bump the `relayer` submodule here and deploy; the signing
workflow's byte-verification step then confirms
`https://relayer.onym.app/manifest.json` serves exactly the committed
bytes.

## Discovery provider

`discovery.onym.app` is live, but nothing in this repository serves it
directly — the signed [discovery](seats/discovery-static-ed25519.md)
provider is published by `onym-discovery`'s own manual deploy workflow
([#4](https://github.com/onymchat/onym-discovery/pull/4), merged; the
genesis publish has run and the live catalog is at sequence 1). The
`workflow_dispatch` `deploy.yml` builds the reference CLI, then signs
and chains the snapshot onto the previously **published** one. A genesis
publish is an explicit input, not a guess.

Before a byte leaves the runner, the workflow verifies everything
exactly as a client would. It then rsyncs the static tree to
`/var/www/discovery` on the **same droplet**, idempotently installs a
Caddy vhost, and only then upserts the grey-cloud DNS record. A mid-run
failure therefore never leaves a public name pointing at a
half-configured host.

Signing seeds (`DISCOVERY_OPERATOR_SEED` and the courier/Blossom seat
seeds) live as Actions secrets, with a `skip_signing` path for operators
who sign locally instead. The job runs in a `production` environment
gated by required reviewers.

Two operational couplings with this repository's deploy:

- **A deploy from this repository sweeps the discovery vhost away.**
  `deploy.sh` rsyncs with `--delete`, which removes the Caddy compose
  override the discovery deploy installed. After an `onym-infra`
  deploy, re-dispatch the `onym-discovery` deploy workflow — it is
  self-healing on re-run and puts the vhost back.
- **The vhost step restarts the shared proxy.** Installing or
  restoring the override recreates the Caddy container that fronts
  every `onym.app` vhost, so open Nostr `wss://` connections drop and
  clients must reconnect. The workflow announces the recreation in its
  log before issuing it.

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
