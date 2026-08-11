# Discovery

Signed, replaceable catalogs for finding compatible instances of every
seat, without becoming a gatekeeper.

**Contract:** [`discovery/Discovery.md`](https://github.com/onymchat/onym-system/blob/main/discovery/Discovery.md)
(the contract as written is proposed architecture and unimplemented)

What exists today is the mechanism, not the seat: four GitHub release
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

## Server manifests

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

## `contracts-manifest.json`

Cumulative — the union of every historical release's contracts, not just
the latest tag — so old and new deployments stay allowlisted together.

## `authorities.json`

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

## Gap

Signature verification is **soft** today. The fetcher accepts an optional
detached `.sig` and checks it when present; absence is not fatal, because
the release-signing pipeline is not live. Until it is, the only control
protecting these catalogs is who can publish releases on those
repositories. Restrict that accordingly.
