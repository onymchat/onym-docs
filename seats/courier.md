# Courier

Moves opaque envelopes and opaque, content-addressed bytes without ever
reading them. Two boundaries — small messages, media blobs — each
technology-free at the contract level: neither requires a particular
event format, socket protocol, relay network, hash, storage engine, or
addressing syntax.

**Contracts:** [`message/UI-Message.md`](https://github.com/onymchat/onym-system/blob/main/message/UI-Message.md)
(small messages) · [`blob/UI-Blob.md`](https://github.com/onymchat/onym-system/blob/main/blob/UI-Blob.md)
(media blobs)
**Implementations:** [Nostr/Blossom](courier-nostr.md) — the current
implementation, and the only one with running code. A
[classical database-backed courier](courier-classical.md) is planned:
no profile text exists yet.

A concrete implementation of the message boundary may use federated
relays, store-and-forward queues, peer-to-peer links, privacy
networks, mesh radio, or another mechanism that satisfies the
contract's opacity, addressing, delivery, receipt, and error
semantics. A concrete implementation of the blob boundary may use
content-addressed HTTP storage, an object network, an institutional
archive, peer-to-peer transfer, local mesh storage, or another
mechanism that satisfies its integrity, opacity, availability,
receipt, and error semantics. Nostr relays and Blossom servers are
today's answer to each — not a requirement either contract states.

## The roles, kept apart on purpose

| Role | What it controls — and only that |
|---|---|
| **UI owner** | Presentation, local orchestration, billing integration, supported profiles, release distribution. |
| **Application-protocol author** | Encrypted inner-envelope formats (message side) or media encoding, encryption, and content-addressing (blob side), sender authentication, replay rules, meaning. |
| **Adapter author** | Maps the common Onym port to one concrete wire technology. |
| **Courier operator** | Endpoints, capacity, routing, retention, moderation, availability, privacy practice, and its own payment model. |
| **User or group** | Which courier(s) to use, and whether to combine several. |

One organization may occupy several roles, but their authorities stay
separate. A user can change UI without needing a new courier. A UI can
use a compatible third-party courier without importing its domain
model. A courier can serve several frontends without learning or
enforcing their group rules.

Each boundary has two interfaces: a local **UI/application ↔ adapter**
port carrying opaque envelopes or blobs, opaque addresses, and typed
outcomes; and a network **adapter ↔ operator** implementation profile
defining framing, connection behavior, access control, payment
refusal, limits, and acknowledgements. Neither interface may expose
plaintext, identity-vault secrets, attachment decryption keys, or
notary witnesses.

## The common surface

Whatever the wire technology underneath, every message-side profile
exposes the same operations:

| Operation | What it does |
|---|---|
| `connect` | Lifecycle intent against a verified route — not proof every endpoint is usable. |
| `publish` / `subscribeTopic` | Many-publisher, many-subscriber delivery over an opaque topic address. |
| `sendInbox` / `subscribeInbox` | Sender-to-one-recipient delivery over an opaque, recipient-derived address. |
| `unsubscribe` / `disconnect` | Local closure, with best-effort remote close. |
| `queryOutcome` | Per-operation or per-message outcome, when the profile supports it. |

And every blob-side profile exposes:

| Operation | What it does |
|---|---|
| `connect` | Per-provider readiness against a verified route and scoped access context. |
| `uploadBlob` / `downloadBlob` | Sealed-blob upload; download that stays explicitly `unverified` until the content reference checks out. |
| `probeBlob` | Availability and provider-side observations for a reference, without downloading it. |
| `deleteBlob` / `replicateBlob` | Provider-scoped erasure; copying already-verified bytes to new providers. |
| `queryOutcome` | Per-operation, per-reference, or per-provider outcome, when supported. |

## Why the boundary is opaque both ways

A courier operator is meant to see routing addresses, timing, and byte
counts — never plaintext, never which application protocol produced
an envelope, never a notary witness. That's what makes couriers
replaceable: an operator that can't read what it moves also can't
hold a UI, a group, or a user's history hostage to its own
continued operation. The cost is symmetric — a UI that wants delivery
guarantees stronger than "opaque bytes moved" has to build that
guarantee itself, on top of what the courier confirms, not underneath
it.

## What this seat admits it can't promise

- **A courier's receipt only proves what it confirms right now.** It
  is not a durability guarantee, and it is not evidence a message or
  blob will still be retrievable later — retention is a declared
  policy each profile states, not a property of the abstract
  contract.
- **Payment refusal must stay distinct from every other failure
  mode** — authentication, rate limiting, invalid framing, network
  failure — and a profile that conflates them has not met the
  contract, whatever its wire format looks like.
- **A second transport technology has to implement the same abstract
  suite without pretending to be the first.** Two profiles that both
  happen to move bytes over WebSockets are not compatible just
  because of that; compatibility means a requester gets the same
  opacity, delivery, and receipt guarantees from either one.

## Next steps

- [Nostr/Blossom](courier-nostr.md) — the one implementation that
  exists today, with running code and a long, honest gap list.
- [Classical (planned)](courier-classical.md) — a conventional
  client-server database-backed courier, named as future work; no
  profile text or code exists yet.
- [Discovery](discovery.md) — how clients learn which courier
  endpoints to actually use, and the signed manifests both current
  couriers already publish.
- [Identity](identity.md) — owns the keys that seal envelopes and
  blobs before they ever reach this boundary; the courier never sees
  them.
