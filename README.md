# Onym seats

Onym splits the powers a normal service concentrates — naming the user,
presenting the interface, carrying the data, validating shared state,
judging conduct — into independently ownable **seats**. One rule governs
all of them:

> A component may exercise the minimum authority required for its role,
> and it may earn only when a user or group chooses it for that role.

Every seat is specified twice in
[`onym-system`](https://github.com/onymchat/onym-system): an abstract
contract, technology-free, and one or more implementation profiles that
map it onto a concrete stack. This book's seat pages document the
abstract contracts; each one links onward to whichever implementations
exist for it — running code, a merged spec, or just a named plan.

[**Open the visual seat map →**](seat-map.md)

The map shows the independently ownable roles inside every documented
seat. A company may occupy several roles, but doing so never merges the
authority each role is allowed to exercise.

## A user's path through them

**[Identity](seats/identity.md)** comes first, because everything else
depends on it and nothing else can substitute for it. Before a person
sends a message or joins a group, a vault on their device produces a
root secret only they hold, and derives from it every purpose-specific
key their activity will need. No server issues this identity and none
can revoke it — losing the root is the only way to lose it.

**[Discovery](seats/discovery.md)** is how that person finds anyone to
talk to next. Every other seat is served by independent operators —
couriers, notaries, moderation authorities — and someone has to
publish the list of who's running what. Discovery makes that list
signed and swappable: a person can verify a catalog, combine several,
or skip catalogs entirely and import an instance's manifest by hand.
No provider's absence from a catalog ever blocks direct use.

**[Courier](seats/courier.md)** carries what they actually send: small
encrypted messages and media blobs, moved by an operator who can see
routing addresses, timing, and byte counts but never plaintext. That
metadata can still reveal patterns, and an unavailable courier can
interrupt delivery or make retained blobs unavailable. The boundary's
promise is narrower: the operator cannot read the content, and clients
can replace it without changing the application protocol.

**[Notary](seats/notary.md)** enters once more than one person needs
to agree on something durable — a group's membership, whose turn it is
to act, what the current state actually is. Rather than trust whoever
happens to keep that state, the group pins a referee that accepts a
change only when it arrives with a zero-knowledge proof that the
change was allowed. The notary checks math, never people, and a
group's choice of notary is sticky: nothing can quietly move a group
to a different one later.

**[Moderation](seats/moderation.md)** is the seat nobody wants to need
and everybody benefits from having. At the moment someone joins an
interface, they consent to one specific authority under its exact
published terms — before any dispute exists. If abuse happens later,
that authority alone can open a case. A human decides by default; an
authority may instead use a local model only when its manifest declares
that autonomous mode before the user consents. Silence can never turn
into a sanction. No platform-wide trust-and-safety team, no authority
with power over people who never agreed to it.

**[Backup](seats/backup.md)** protects what a person has already
built. A phone gets lost, stolen, or replaced, and an operator holding
a sealed, unreadable copy of a device's history is what keeps that from
being total loss — an operator who counts bytes and enforces a
deadline, and who can't read what it stores or hand it to anyone but
the person who sealed it.

**[Charity](seats/charity.md)** is a more specialized journey than the
others, layered on top of what came before: a donor wants proof their
money reached a real program, and a beneficiary needs help without
handing over their whole identity to get it. The seat composes an
operator, a credential issuer, a financial provider, a notary, and an
auditor as separately replaceable roles, so a beneficiary can prove
they're *eligible* for aid without publishing who they *are*. An issuer
or regulated provider may still require private identity evidence under
its own declared purpose and retention terms.

## What exists today

Nothing here is production-grade: running components are alpha and
unaudited, and several load-bearing pieces remain open work. As of
21 August 2026, the shortest honest status map is:

| Seat | Most mature implementation | Status |
|---|---|---|
| [Identity](seats/identity.md) | BIP-39 | Running in both clients; major capability, rotation, and conformance gaps |
| [Discovery](seats/discovery.md) | Static snapshot / Ed25519 | Signed provider and client packages exist; shipping clients still prefer legacy unsigned assets |
| [Courier](seats/courier.md) | Nostr/Blossom | Running; material receipt, verification, payment, and replication gaps |
| [Notary](seats/notary.md) | Stellar/Soroban | Running on Stellar testnet; not production-audited |
| [Moderation](seats/moderation.md) | DeviceCheck / device recall | iOS and Android services run; Android recovery and platform-access gaps remain |
| [Backup](seats/backup.md) | Object-HTTP | Running in free mode; paid path unexercised, conformance fixtures unwritten |
| [Charity](seats/charity.md) | BNB Chain | Specification merged; no code; Stellar, Cardano, and Solana remain plans |

Each seat page links to its implementation pages, where the status and
limitations are described in full.

## Named, not yet built

Contract text exists for a handful of seats this book doesn't document
yet, because no implementation of any kind exists to describe:
**recovery trustee** (who recovers an identity when its root secret is
lost, distinct from a backup restore), **audit**, **arbitration**,
**lead generation**, **acquisition**, **sponsor**, and **recruitment**.
The **interface** seat — the app vendor's own role — has client
scaffolds in [`onym-ios`](https://github.com/onymchat/onym-ios) and
[`onym-android`](https://github.com/onymchat/onym-android) but doesn't
yet meet its own contract. The **bank** seat is an open pull request
against `onym-system`; **association naming** is named in that repo's
overview but has no contract text on any open branch yet.

[Deployment](deployment.md) brings the server-side seats up on one box.
