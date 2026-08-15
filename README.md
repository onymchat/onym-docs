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

This book documents the seats that have **running reference code** today.
It is not the specification — it is the map from a contract to the repo,
the wire surface, and the command that starts it.

## Implemented

| Seat | Reference implementation | Live at |
|---|---|---|
| [Moderation](seats/moderation.md) | [`onym-moderation`](https://github.com/onymchat/onym-moderation) — `authority/` + `apple/` | `authority.onym.app`, `moderation.onym.app` |
| [Notary](seats/notary.md) | [`onym-contracts`](https://github.com/onymchat/onym-contracts) (5 Soroban contracts) + [`onym-relayer`](https://github.com/onymchat/onym-relayer) | Stellar testnet, `relayer.onym.app` |
| [Courier](seats/courier.md) | strfry (Nostr) + blossom-server, wired in [`onym-infra`](https://github.com/onymchat/onym-infra) | `nostr.onym.app`, `blossom.onym.app` |
| [Identity](seats/identity.md) | [`onym-sdk-swift`](https://github.com/onymchat/onym-sdk-swift) / [`onym-sdk-kotlin`](https://github.com/onymchat/onym-sdk-kotlin), consumed by the clients | on device |
| [Discovery](seats/discovery.md) | Five GitHub release assets across three repos (operational) + [`onym-discovery`](https://github.com/onymchat/onym-discovery) reference CLI for the signed-catalog profile | `releases/latest/download/*` (what clients read), `discovery.onym.app` (signed catalog) |

[Deployment](deployment.md) brings the server-side seats up on one box.

## Not implemented

Contract only, no code in any Onym repository: **recovery trustee**,
**backup**, **charity**, **audit**, **arbitration**, **lead generation**,
**acquisition**, **sponsor**, **recruitment**. The **interface** seat has
client scaffolds ([`onym-ios`](https://github.com/onymchat/onym-ios),
[`onym-android`](https://github.com/onymchat/onym-android)) but does not
yet meet its contract. The **bank** seat is an open pull request against
`onym-system`; the **association naming** seat is named in that repo's
overview but has no contract text on any open branch yet.

Nothing here is production-grade: alpha, unaudited, and several
load-bearing pieces are open work.
