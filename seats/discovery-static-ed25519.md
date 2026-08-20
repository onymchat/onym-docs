# Discovery — Static Snapshot / Ed25519

The only implementation profile that exists today, merged with running
reference code — and, right now, two realities that coexist. The
**operational path**, what every shipping client actually reads today,
is a handful of unsigned GitHub release assets, documented first
below. The **signed-catalog path** is this merged profile itself, with
a reference implementation, merged client packages, and a live
provider at `discovery.onym.app`; it is what the release assets are
migrating onto.

**Status:** Running transition. The signed provider and client packages
exist, while shipping clients still prefer the legacy unsigned assets.

**Profile:** [`discovery/Discovery-Static-Ed25519.md`](https://github.com/onymchat/onym-system/blob/main/discovery/Discovery-Static-Ed25519.md)
(merged in [PR #28](https://github.com/onymchat/onym-system/pull/28))
— implements the abstract [Discovery](discovery.md) contract. The
profile's §11 is the single source of truth for implementation status.
**Code:** [`onym-discovery`](https://github.com/onymchat/onym-discovery)
(reference CLI + conformance fixtures)

## Today's operational path: release assets

What runs today is a mechanism, not the seat: five GitHub release
assets that clients fetch at
`https://github.com/onymchat/<repo>/releases/latest/download/<asset>`.
A release asset, **not** a path in the tree — editing `main` changes
nothing any user sees. That's deliberate: these files decide who
serves and who judges, so they move on a reviewed, dated, revertible
artifact.

| Asset | Repo | Consumer |
|---|---|---|
| `relayers.json` | `onym-relayer` | clients pick a Soroban relayer |
| `nostr-relays.json` | `onym-relayer` | clients connect to **all** listed |
| `blossom-servers.json` | `onym-relayer` | clients use the **first** listed |
| `contracts-manifest.json` | `onym-contracts` | the relayer's contract allowlist |
| `authorities.json` | `onym-authorities` | the iOS moderation picker |

### Server manifests

```json
{ "version": 1, "relays": [ { "name": "Onym Official", "url": "wss://nostr.onym.app", "isDefault": true } ] }
```

`scripts/validate-server-manifest.py` enforces the shape at release
time: Nostr URLs `wss://`/`ws://`, Blossom `https://`/`http://`, at
most one `isDefault`. `relayers.json` entries carry `name`, `url` (an
origin) and `networks`; the request body still selects the Stellar
network.

Third-party operators add themselves by PR against the tracked file;
the release workflow validates HTTPS URLs, unique origins and
supported networks before publishing.

### `contracts-manifest.json`

Cumulative — the union of every historical release's contracts, not
just the latest tag — so old and new deployments stay allowlisted
together.

### `authorities.json`

```json
{
  "authorities": [{
    "componentId": "onym:component:<id>",
    "name": "Shown in the picker",
    "manifestURL": "https://<host>/manifest.json",
    "apiBaseURL": "https://<host>",
    "operatorPublicKeyBase64": "<base64 of 32 raw bytes>"
  }]
}
```

The operator key is duplicated here **on purpose**. The client
verifies verdicts against this key, not the one in the fetched
manifest, so an attacker who can substitute a manifest cannot also
substitute the key it is checked against. That property holds only if
the value was obtained out of band — from the operator, or from the
running service's `/health`, never by fetching the manifest.

Note the encoding split: base64 of raw bytes here, `onym:key:<hex>`
everywhere else.

```sh
python3 -c "import base64,sys; print(base64.b64encode(bytes.fromhex(sys.argv[1])).decode())" <hex>
```

**Publishing the first release turns moderation on** for every user on
a build that reads this. The consent gate blocks only when the
directory yields entries; until then the app runs unmoderated. It is a
product change, not a registration step.

Adding an authority: get the key out of band → fetch their manifest
and confirm `operator` and `componentId` match → read the published
terms their class `definition` URLs point at → PR → release.

### The gap that motivates everything below

Signature verification on these assets is **soft** today. The fetcher
accepts an optional detached `.sig` and checks it when present;
absence is not fatal, because the release-signing pipeline is not
live. Until it is, the only control protecting these catalogs is who
can publish releases on those repositories. Restrict that accordingly.

## The signed-catalog path

This profile replaces trust-the-host with verify-the-bytes. A provider
is just an Ed25519 keypair and a static file host: it publishes a
self-signed `manifest.json` naming its catalogs, and each catalog is a
signed snapshot — the full list in one file, which your client
downloads whole and filters **locally**, so no server ever learns what
you searched for. Snapshots chain: each carries a sequence number and
the hash of its predecessor's exact bytes, so a provider that rolls
back, forks, or quietly rewrites history produces cryptographic
evidence against itself. The first time you add a provider, your
client shows the operator key's fingerprint and **pins it**
(trust-on-first-use); after that, a file signed by any other key is an
alarm, never a silent rotation. And catalogs expire on a hard ceiling,
so a compromised or abandoned listing ages out instead of being
recommended forever.

What exists, honestly:

- **A reference implementation runs**:
  [`onym-discovery`](https://github.com/onymchat/onym-discovery) is a
  Rust CLI that signs, verifies, and chains manifests and snapshots,
  and publishes the byte-pinned conformance fixtures clients must
  match — after the merged gap-closure sweep
  ([#3](https://github.com/onymchat/onym-discovery/pull/3)), most of
  the profile's §10 vectors are published as fixtures and the rest
  (the chain-behavior cases) are covered by in-repo tests, with the
  privacy trace discharged as a client obligation — plus deployment
  templates and a publish runbook.
- **A provider is live**: `discovery.onym.app` serves a signed
  provider manifest and the `onym-services` catalog, with its
  inclusion policy and privacy profile pinned by digest. Its operator
  fingerprint is published below.
- **Client packages are merged**: iOS
  ([#244](https://github.com/onymchat/onym-ios/pull/244)–[#247](https://github.com/onymchat/onym-ios/pull/247))
  and Android
  ([#204](https://github.com/onymchat/onym-android/pull/204)–[#208](https://github.com/onymchat/onym-android/pull/208))
  — fetching, TOFU key pinning, chain verification, source management,
  and the consent UI, wired behind the legacy fetchers as a fallback.

## Operator fingerprints

When you add a discovery provider, your client shows you an operator
key fingerprint and asks you to confirm it before pinning
(trust-on-first-use). That confirmation is only as good as the value
you compare it against — so here are the reference operators'
fingerprints, published out of band from the services themselves.
**Compare the fingerprint on your app's TOFU screen against the value
below.** If they match, confirm and the key is pinned; if they don't,
stop — you are not talking to the operator this page describes.

| Service | Fingerprint | Operator key |
|---|---|---|
| `discovery.onym.app` (discovery) | `4d:a9:ec:c9:e8:6f:6e:97` | `onym:key:42b0da001104dd03052c7feddab9520c920c9e40d11b245c46c27cf6be853f24` |
| `relayer.onym.app` (notary) | `28:77:a5:5c:c4:ae:20:ec` | `onym:key:8c836293161a3ee2e4c2e338851d88289a2db494efc6342d9fb7ac0c516936ad` (manifest `validUntil` 2027-08-14) |

The live `onym-services` catalog also lists these operators for the
other seats. Their manifests are indexed by the catalog, so the pinned
discovery key already protects them — but the keys are repeated here
for out-of-band comparison:

| Service | Fingerprint | Operator key |
|---|---|---|
| `onym-authority` (moderation) | `fd:92:53:ed:1f:1e:35:7d` | `onym:key:bdec68a8440f36591dd822748f86fee3582794b3d20445b06953db6f266f3dca` |
| `onym-courier` (transport.message) | `6b:14:cd:ea:7e:95:be:60` | `onym:key:92500a19c43193c8945aa91b94878b0c986f2c4da500de4c2caea29103aaa84f` |
| `onym-blossom` (blob.storage) | `4a:e9:35:23:ef:49:b6:00` | `onym:key:e446f2b18e9f75e13397ebdff0f2e40610c9e745aa11308b04c8b607f7dea094` |

A fingerprint is the first 8 bytes, colon-separated hex, of the
SHA-256 of the key's 32 raw public-key bytes. Every value above was
verified on **2026-08-15** against the manifests actually served at
`https://discovery.onym.app/manifest.json`,
`https://relayer.onym.app/manifest.json`, and the operators listed in
`https://discovery.onym.app/catalogs/onym-services.json`, with the
fingerprints recomputed from the served keys. If this page and your
TOFU screen ever disagree, treat the disagreement itself as the signal
and ask before pinning.

## Honest status

The profile's own gaps section is candid, and this page will not
outrun it:

- **The release assets have not migrated.** `discovery.onym.app` is
  live and serves a signed catalog, but the shipping clients still
  read the release assets above as their operational path; migrating
  them onto the signed catalogs is explicitly listed as remaining
  work.
- **Some checks live only in the reference CLI.** Duplicate-key
  rejection, the detached-`.sig` verify path, and cross-catalog
  equivocation / source-conflict detection are implemented and
  fixtured in `onym-discovery`, but neither client package runs them
  yet. The clients also still approximate an expired provider manifest
  as a plain refresh failure, don't surface entry-vs-manifest field
  conflicts under their proper error, and leave several of the
  profile's error codes unreachable.
- **The intermediate-fetch continuity walk is implemented nowhere.**
  Every implementation degrades a forward jump straight to
  accept-with-note without first trying the retained-sibling fetches
  the profile's §6 requires, so a provably broken chain hidden behind
  a jump is indistinguishable from a retention failure. The profile's
  §11 enumerates this and the smaller remainders honestly — it is the
  single place to check before trusting any status claim, including
  this page's.

## Next steps

- [Discovery](discovery.md) — the technology-free contract this
  profile implements.
- [Notary](notary.md) — the relayers today's `relayers.json` points
  at, and the operator manifest the live catalog already indexes.
- [Moderation](moderation.md) — the authorities `authorities.json`
  feeds into the iOS picker.
- [Deployment](../deployment.md) — how the reference deployment brings
  the server-side seats up.
