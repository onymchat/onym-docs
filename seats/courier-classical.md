# Courier — Classical

*Seat implementation page, draft 0.1 — 20 August 2026. Specification:
not written. Code: none — see [Honest status](#honest-status).*

**Profile:** must be written — a `message/UI-Message-Classical.md` and
a `blob/UI-Blob-Classical.md` in `onym-system`, binding the
[`message/UI-Message.md`](https://github.com/onymchat/onym-system/blob/main/message/UI-Message.md)
and [`blob/UI-Blob.md`](https://github.com/onymchat/onym-system/blob/main/blob/UI-Blob.md)
boundaries to a conventional client-server backend: an ordinary
message queue and a database-backed object store, instead of a Nostr
relay and a Blossom server.
**Code:** none.

This page does not summarize a profile, because there is no profile.
It states the case for the binding and what it would have to decide.

## Why a second profile at all

The [Nostr/Blossom implementation](courier-nostr.md) works, but it
ties every courier operator to running (or trusting) relay and
Blossom-server software from outside the Onym project, with protocol
quirks — event kinds, NIP-42 auth, BUD-01/02/07/11/12 — that an
operator otherwise unfamiliar with the Nostr ecosystem has to learn.
A classical, database-backed courier would let an operator stand up a
conforming courier on infrastructure they already run and understand:
an authenticated queue for small messages, an authenticated
object store for blobs, both fronted by an ordinary HTTP or gRPC API.

The abstract contracts already anticipate this. `UI-Message.md`
names "federated relays, store-and-forward queues, peer-to-peer
links, privacy networks, mesh radio" as valid alternatives to Nostr,
and `UI-Blob.md` names "content-addressed HTTP storage, an object
network, an institutional archive" as valid alternatives to Blossom —
a plain database-backed store sits squarely inside both lists. Being
plainer than Nostr is a feature here, not a compromise: fewer
protocol-specific quirks for a second implementation to get wrong,
which is also what would make it a genuine interoperability test of
the abstract boundary rather than a second copy of the first profile's
choices.

## What the profile would have to decide

Nothing below is answered. These are the questions
[`Nostr/Blossom`](courier-nostr.md) already had to answer once, that a
classical profile would answer differently on purpose:

- **Addressing**, in place of Nostr pubkeys and event IDs — an opaque
  routing address the contract requires either way, but derived and
  stored how, in a relational or document store.
- **Delivery and receipts** — the message contract's `queryOutcome`
  and per-courier outcome semantics, in place of relay `OK` messages;
  a classical backend can plausibly do *better* here (structured
  receipts, per-recipient delivery state) rather than merely matching
  Nostr's gaps.
- **Access control and payment refusal**, in place of NIP-42 auth and
  Blossom's Nostr-event-signed authorization — both contracts require
  payment refusal to be distinct from authentication and rate
  limiting, which a classical HTTP API can express directly as
  distinct status codes.
- **Fan-out policy** — whether a classical courier follows Nostr's
  "every listed relay" rule, Blossom's "first listed server" rule, or
  states its own, and how [Discovery](discovery.md)'s manifest format
  would need to grow a new profile-declared fan-out field to carry it.
- **Retention and deletion semantics** a conventional database makes
  easy to offer (TTLs, explicit per-message erasure) that the current
  profile's gap list gestures at but doesn't specify.

## Honest status

- **Nothing exists.** No profile document, no adapter, no server, no
  fixtures. `onym-system` holds no `*-Classical.md` file under
  `message/` or `blob/`.
- **The abstract contracts don't block this.** Both `UI-Message.md`
  and `UI-Blob.md` explicitly reserve room for a second technology; a
  classical profile needs its own implementation profile ID under
  each, and conformance against the shared fixture suites named in
  each contract's acceptance criteria — not permission to diverge from
  Nostr/Blossom's choices where they were Nostr/Blossom-specific
  rather than boundary-required.
- **No operator or timeline is committed.** This page exists to make
  the shape of the work legible before anyone starts it, not to
  promise it's scheduled.

## Next steps

- [Courier](courier.md) — the technology-free contract this would
  bind, on both the message and blob side.
- [Nostr/Blossom](courier-nostr.md) — the one implementation that
  exists today, and the profile whose gap list this page's open
  questions were drawn against.
- [Discovery](discovery.md) — the manifest format a classical courier
  would need to declare itself into, alongside the fan-out rule it
  chooses.
