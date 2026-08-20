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
routing addresses and byte counts but never plaintext. A courier
doesn't know who's talking, doesn't know what they're saying, and
can't hold a conversation hostage to its own continued operation —
replacing one is the point.

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
that authority alone can open a case, and only a human decision, never
silence, can turn into a sanction. No platform-wide trust-and-safety
team, no authority with power over people who never agreed to it.

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
they're *eligible* for aid without ever proving who they *are*.

## What exists today

Nothing here is production-grade: alpha, unaudited, and several
load-bearing pieces are open work. Each seat page states plainly what
runs, what's merely specified, and what's still a plan — the honest
answer differs by seat and sometimes by implementation within a seat,
so this book doesn't summarize it in one table anymore. Start from the
seat page and follow its own "Implementations" line.

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
