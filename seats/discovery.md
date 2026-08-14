# Discovery

How does your app find a relayer to submit proofs through, a moderation
authority to consent to, a Nostr relay to speak over? Someone has to
publish that list — and whoever publishes it holds real power, because
a list decides who serves and who judges. The discovery seat exists to
make that power **signed, inspectable, and replaceable**: catalogs you
can verify, providers you can swap, and a guarantee that absence from
every catalog never blocks you from using an instance you found
yourself.

**Contract:** [`discovery/Discovery.md`](https://github.com/onymchat/onym-system/blob/main/discovery/Discovery.md)
· profile: [static snapshot / Ed25519](https://github.com/onymchat/onym-system/blob/main/discovery/Discovery-Static-Ed25519.md)
(merged, with running reference code)
**Code:** [`onym-discovery`](https://github.com/onymchat/onym-discovery)
(reference CLI + conformance fixtures)

Two realities coexist on this page, and both are real. The
**operational path** — what every shipping client reads today — is a
handful of unsigned GitHub release assets, documented first below. The
**signed-catalog path** is the merged implementation profile with a
reference implementation and client packages in review; it is what the
release assets migrate onto, and it is documented after.

## Today's operational path: release assets

What runs today is the mechanism, not the seat: four GitHub release
assets that clients fetch at
`https://github.com/onymchat/<repo>/releases/latest/download/<asset>`.
A release asset, **not** a path in the tree — editing `main` changes
nothing any user sees. That is deliberate: these files decide who serves
and who judges, so they move on a reviewed, dated, revertible artifact.

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

`scripts/validate-server-manifest.py` enforces the shape at release time:
Nostr URLs `wss://`/`ws://`, Blossom `https://`/`http://`, at most one
`isDefault`. `relayers.json` entries carry `name`, `url` (an origin) and
`networks`; the request body still selects the Stellar network.

Third-party operators add themselves by PR against the tracked file; the
release workflow validates HTTPS URLs, unique origins and supported
networks before publishing.

### `contracts-manifest.json`

Cumulative — the union of every historical release's contracts, not just
the latest tag — so old and new deployments stay allowlisted together.

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

The operator key is duplicated here **on purpose**. The client verifies
verdicts against this key, not the one in the fetched manifest, so an
attacker who can substitute a manifest cannot also substitute the key it
is checked against. That property holds only if the value was obtained
out of band — from the operator, or from the running service's `/health`,
never by fetching the manifest.

Note the encoding split: base64 of raw bytes here, `onym:key:<hex>`
everywhere else.

```sh
python3 -c "import base64,sys; print(base64.b64encode(bytes.fromhex(sys.argv[1])).decode())" <hex>
```

**Publishing the first release turns moderation on** for every user on a
build that reads this. The consent gate blocks only when the directory
yields entries; until then the app runs unmoderated. It is a product
change, not a registration step.

Adding an authority: get the key out of band → fetch their manifest and
confirm `operator` and `componentId` match → read the published terms
their class `definition` URLs point at → PR → release.

### The gap that motivates everything below

Signature verification on these assets is **soft** today. The fetcher
accepts an optional detached `.sig` and checks it when present; absence
is not fatal, because the release-signing pipeline is not live. Until it
is, the only control protecting these catalogs is who can publish
releases on those repositories. Restrict that accordingly.

## The signed-catalog path

The merged
[static-snapshot / Ed25519 profile](https://github.com/onymchat/onym-system/blob/main/discovery/Discovery-Static-Ed25519.md)
replaces trust-the-host with verify-the-bytes. A provider is just an
Ed25519 keypair and a static file host: it publishes a self-signed
`manifest.json` naming its catalogs, and each catalog is a signed
snapshot — the full list in one file, which your client downloads whole
and filters **locally**, so no server ever learns what you searched
for. Snapshots chain: each carries a sequence number and the hash of
its predecessor's exact bytes, so a provider that rolls back, forks, or
quietly rewrites history produces cryptographic evidence against
itself. The first time you add a provider, your client shows the
operator key's fingerprint and **pins it** (trust-on-first-use); after
that, a file signed by any other key is an alarm, never a silent
rotation. And catalogs expire on a hard ceiling, so a compromised or
abandoned listing ages out instead of being recommended forever.

What exists, honestly:

- **The contract and profile are merged** in `onym-system`
  ([PR #28](https://github.com/onymchat/onym-system/pull/28)); the
  profile's §11 is the single source of truth for implementation
  status.
- **A reference implementation runs**:
  [`onym-discovery`](https://github.com/onymchat/onym-discovery) is a
  Rust CLI that signs, verifies, and chains manifests and snapshots,
  and publishes the first conformance fixtures clients must match
  byte-for-byte, plus deployment templates for a future provider.
- **Client packages are written and in review**: iOS
  ([#244](https://github.com/onymchat/onym-ios/pull/244)–[#247](https://github.com/onymchat/onym-ios/pull/247))
  and Android
  ([#204](https://github.com/onymchat/onym-android/pull/204)–[#208](https://github.com/onymchat/onym-android/pull/208))
  — fetching, TOFU key pinning, chain verification, source management,
  and the consent UI, wired behind the legacy fetchers as a fallback.

## Honest limits

The profile's own gaps section is candid, and this page will not
outrun it:

- **No provider is deployed.** `discovery.onym.app` serves nothing yet:
  no operator keys, no signed catalog. The release assets above remain
  the only operational path, and migrating them onto signed catalogs is
  explicitly listed as remaining work.
- **The policy documents are unwritten.** Every catalog must pin an
  inclusion/ranking policy and a privacy profile by digest; those
  documents do not exist yet.
- **Client packages trail the spec in places.** The open PRs verify
  signatures, pin keys, and detect rollbacks and forks, but neither
  client yet rejects a future-dated snapshot, filters catalogs by
  audience, or tells the user how many entries it silently skipped.
- **Some specified behavior is implemented nowhere yet** — the
  forward-jump continuity walk, the policy-transition grace window,
  and several disclosure obligations (every current decoder even skips
  entries carrying the profile's warning `status` field). The
  profile's §11 enumerates them honestly; the fixtures that will prove
  them (§10 items 6–14) are also still to be written.

## Next steps

- [Notary](notary.md) — the relayers today's `relayers.json` points at,
  and the operator manifests the signed catalogs will index.
- [Moderation](moderation.md) — the authorities `authorities.json`
  feeds into the iOS picker.
- [Deployment](../deployment.md) — how the reference deployment brings
  the server-side seats up.
