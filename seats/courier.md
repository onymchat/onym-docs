# Courier

Moves encrypted envelopes and media without reading them. Two boundaries,
two profiles, two off-the-shelf servers.

| | Small messages | Media blobs |
|---|---|---|
| Contract | [`UI-Message.md`](https://github.com/onymchat/onym-system/blob/main/message/UI-Message.md) | [`UI-Blob.md`](https://github.com/onymchat/onym-system/blob/main/blob/UI-Blob.md) |
| Profile | [Nostr](https://github.com/onymchat/onym-system/blob/main/message/UI-Message-Nostr.md) | [Blossom](https://github.com/onymchat/onym-system/blob/main/blob/UI-Blob-Blossom.md) |
| Server | `dockurr/strfry` | `ghcr.io/hzrd149/blossom-server` |
| Live | `wss://nostr.onym.app` | `https://blossom.onym.app` |
| Client code | `onym-ios`, `onym-android` | same |

There is no Onym-written courier. The seat is satisfied by a stock relay
under a pinned configuration — replaceability is the point, so the
interesting code is entirely on the client side.

## Relay configuration

[`onym-infra/strfry/strfry.conf`](https://github.com/onymchat/onym-infra/blob/main/strfry/strfry.conf)
is the deployed policy:

```
maxEventSize            65536
maxWebsocketPayloadSize 131072
maxNumTags              2000       maxTagValSize  1024
maxFilterLimit          500        maxSubsPerConnection 20
rejectEventsNewerThanSeconds  900  (older than) 94608000
ephemeralEventsLifetimeSeconds 300
```

Blossom runs with `DATA_DIR=/app/data`, `PORT=3000`, behind Caddy. Neither
container publishes a host port.

## Fan-out rules

The two manifests differ deliberately (see [Discovery](discovery.md)):

- **Nostr** — clients connect to **every** listed relay.
- **Blossom** — clients upload and download via the **first** listed
  server, so order matters.

Clients seed a hardcoded default at first launch (offline-proof), refresh
from the manifests in the background, and never overwrite a user's own
custom entries.

## Gaps

Both profiles carry long, explicit gap lists. The load-bearing ones:

**Nostr** — receipts expose only `messageId` and aggregate `acceptedBy`,
not per-relay outcomes; publish success is inferred from a local socket
send or a five-second `OK` timeout rather than `OK true`; `OK` reasons are
discarded and `AUTH`/`CLOSED` ignored, so authentication and payment
refusal cannot reach the UI; clients recompute event IDs but do not
consistently perform full Schnorr verification. iOS and Android diverge in
subscription and reconnect behavior.

**Blossom** — each app uses only the first configured server instead of a
replicated `BlobRoute`; the seam exposes only upload and complete
download; uploads omit `X-SHA-256`, use padded rather than URL-safe
Base64, and send the plaintext MIME type for encrypted objects; returned
descriptors are not fully verified.

**Both** — `PaymentRequired`, `SeatEntitlement`, quota accounting, renewal,
revocation and broker registration are unimplemented, and relay/provider
configuration binds no operator, limits, retention, or privacy terms.
Conformance rests on mirrored app tests rather than one fixture suite.

These are interoperability and security gaps, not alternate wire semantics
a third party should copy.
