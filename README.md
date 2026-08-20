# Onym seats

Onym splits the powers a normal service concentrates — naming the user,
presenting the interface, carrying the data, validating shared state,
judging conduct — into independently ownable **seats**. One rule governs
all of them:

> A component may exercise the minimum authority required for its role,
> and it may earn only when a user or group chooses it for that role.

Every seat is specified twice in
[`onym-system`](https://github.com/onymchat/onym-system): an abstract
contract (`X.md`) that is technology-free, and one or more implementation
profiles (`X-<Technology>.md`) that map it onto a concrete stack.

This book documents the seats that have **running reference code**
today, plus — clearly marked as plans — the seats whose build is
designed but not started. It is not the specification — it is the map
from a contract to the repo, the wire surface, and the command that
starts it (or, for a plan, to the work that would).

## Implemented

| Seat | Reference implementation | Live at |
|---|---|---|
| [Moderation](seats/moderation.md) | [`onym-moderation`](https://github.com/onymchat/onym-moderation) — `authority/` + [`apple/`](seats/moderation-ios.md) + [`android/`](seats/moderation-android.md) | `authority.onym.app`, `moderation.onym.app`, `moderation-android.onym.app` |
| [Notary](seats/notary.md) | [Stellar/Soroban](seats/notary-stellar.md): [`onym-contracts`](https://github.com/onymchat/onym-contracts) (5 Soroban contracts) + [`onym-relayer`](https://github.com/onymchat/onym-relayer); a [BNB implementation](seats/notary-bnb.md) is planned, unbuilt | Stellar testnet, `relayer.onym.app` |
| [Courier](seats/courier.md) | [Nostr/Blossom](seats/courier-nostr.md): strfry (Nostr) + blossom-server, wired in [`onym-infra`](https://github.com/onymchat/onym-infra); a [classical database-backed courier](seats/courier-classical.md) is planned, unbuilt | `nostr.onym.app`, `blossom.onym.app` |
| [Identity](seats/identity.md) | [BIP-39](seats/identity-bip39.md): [`onym-ios`](https://github.com/onymchat/onym-ios) / [`onym-android`](https://github.com/onymchat/onym-android) (derivation + `IdentityRepository`), over shared primitives from [`onym-sdk-swift`](https://github.com/onymchat/onym-sdk-swift) / [`onym-sdk-kotlin`](https://github.com/onymchat/onym-sdk-kotlin) | on device |
| [Discovery](seats/discovery.md) | legacy release assets (operational) + [`onym-discovery`](https://github.com/onymchat/onym-discovery) reference CLI for the [Static Snapshot / Ed25519](seats/discovery-static-ed25519.md) signed-catalog profile | `releases/latest/download/*` (what clients read), `discovery.onym.app` (signed catalog) |

[Deployment](deployment.md) brings the server-side seats up on one box.

## Not implemented

Contract only, no code in any Onym repository: **recovery trustee**,
**audit**, **arbitration**, **lead generation**, **acquisition**,
**sponsor**, **recruitment**. The **[backup](seats/backup.md)** seat
has a draft contract and a merged
[object-HTTP implementation profile](seats/backup-object-http.md)
— wire mapping, sealing, restore, erasure, and export are all pinned —
but no adapter, no operator, and no fixtures. The
**[charity](seats/charity.md)** seat has draft contracts, a
[Stellar/Soroban plan](seats/charity-stellar.md), a
[merged BNB Chain specification](seats/charity-bnb.md), an
[adversary-view threat model](seats/charity-adversary.md), and a
[role-binding table](seats/charity-roles.md) documented in this book,
but no code. The **interface** seat has
client scaffolds ([`onym-ios`](https://github.com/onymchat/onym-ios),
[`onym-android`](https://github.com/onymchat/onym-android)) but does not
yet meet its contract. The **bank** seat is an open pull request against
`onym-system`; the **association naming** seat is named in that repo's
overview but has no contract text on any open branch yet.

Nothing here is production-grade: alpha, unaudited, and several
load-bearing pieces are open work.
