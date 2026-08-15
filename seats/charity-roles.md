# Charity roles: who would hold what

*Role-binding page, draft 0.2 — 15 August 2026.*

The abstract contract's first decision is that charity is composed of
**separately replaceable roles** whose identities are **separately
inspectable** (`Charity.md` §1, §3.3). That separation is easy to
state and easy to fake — a diagram proves nothing. This page makes it
operational: for each abstract role, which concrete party holds it in
the reference deployment today, on what evidence, and — for most of
them — the honest answer **unassigned**.

## The binding table

"Evidence" means a signed, inspectable artifact a third party can
check; "planned home" is where the seat pages expect the role to land
first, which binds nobody until a signed record exists.

| Abstract role (`Charity.md` §1) | Concrete party today | Evidence | Status |
|---|---|---|---|
| User application | None conforming. The Onym client scaffolds ([`onym-ios`](https://github.com/onymchat/onym-ios), [`onym-android`](https://github.com/onymchat/onym-android)) contain no charity module implementing the `UI-Charity.md` port. | — | **Unassigned** (planned home: the Onym clients) |
| Charity operator | None. | — | **Unassigned.** No party has published a signed `CharityDeployment`, and this book names no candidate: an operator exists when its signature does. |
| Organization credential issuer | None. | — | **Unassigned.** No issuer, no `TrustPolicy` naming one, no credential schema in any Onym repository. |
| Eligibility issuer | None. | — | **Unassigned.** The merged [BNB profile](https://github.com/onymchat/onym-system/blob/main/charity/UI-Charity-BNB.md) specifies what an issuer must publish (an accumulator root per policy); nobody publishes one. |
| Financial provider | None bound. Fiat rails under the financial provider's own legal authority are the assumed starting point on both chain pages — a role the charity operator may also hold, but then as a *named* second role, not by default — and *assumed* is not *bound*: no `financialBindings` entry exists because no deployment exists. | — | **Unassigned**, and partly **out of scope by design** — the merged BNB profile binds notary and eligibility roles only; the [Stellar page](charity-stellar.md) assumes the same scope and has no profile to bind it yet. |
| Notary | None for charity. The reference relayer operator holds the **notary-seat** operator role on Stellar today, with the signed manifest at `relayer.onym.app/manifest.json` as evidence — but that manifest declares Stellar notary support only. It declares no charity profile, so no charity notary exists. | [`relayer.onym.app/manifest.json`](https://relayer.onym.app/manifest.json) (notary seat only; context: [the Stellar notary page](notary-stellar.md#the-live-operator-manifest)) | **Unassigned** (planned home: the same relayer operator, by adding the charity profile entries its manifest currently, honestly, lacks) |
| Auditor / report issuer | None. | — | **Unassigned.** Fund-flow reports name their issuer by signature; no issuer, no reports. |
| Discovery / association registry | The signed catalog at `discovery.onym.app` runs and lists what operator manifests declare. Since no manifest declares a charity deployment, it lists none — correct behavior, not a gap. | [`discovery.onym.app`](https://discovery.onym.app) (context: [Discovery](discovery.md)) | **Assigned for its own seat; nothing charity-shaped to list yet** |

One row is load-bearing for reading the rest: the only signed
artifact that exists anywhere in this table is the notary-seat
manifest, and it is evidence *because it declines to claim the
charity role*. That is the pattern every other row must follow — a
role is held when a signed record says so, and not one page earlier.

## One party, several roles, named authority each time

The contract permits one party to hold several roles, but every
signed record must state **which authority it exercises**
(`Charity.md` §1). The realistic first deployment concentrates
roles — the reference relayer operator as charity notary, an aid
organization as both charity operator and eligibility issuer. The
separation survives concentration only if the records keep naming the
role, so a reader can verify it record by record:

- the operator manifest lists `seat` and `implementationProfiles` per
  entry — the same key can appear under the notary seat and a charity
  profile, and each entry is a distinct declaration with distinct
  declared powers;
- a `CharityDeployment` names its `operator` key for the operator
  role specifically, and its `notaryBindings`, `eligibilityBindings`,
  and `auditBindings` name their providers separately even when the
  keys coincide;
- an `OrganizationCredential` names its `issuer`, an
  `EligibilityPolicy` names *its* issuers, and a fund-flow report
  names its own `issuer` — three fields that may hold one key three
  times, each a separate claim carrying separate responsibility;
- on-chain, the merged BNB profile (summarized on
  [the BNB page](charity-bnb.md)) enforces the split
  mechanically: operator-attested entrypoints are gated on the admin
  address, claim anchoring is sender-agnostic and proof-authorized —
  so even the concentrated party *cannot* exercise the beneficiary's
  authority, only its own.

The test a skeptical reader should apply to any future deployment:
pick a signed record, and ask which role signed it. If the answer
requires knowing "well, it's all the same organization," the
deployment fails `Charity.md` §3.3 regardless of who owns which keys.

## What this page is for

When a deployment forms, this table is the checklist: each row flips
from **unassigned** to a link to the signed artifact that assigned
it — a manifest entry, a deployment object, a trust policy, a
credential schema. A row without an artifact stays unassigned no
matter what any announcement says.

## Next steps

- [Charity](charity.md) — the roles' abstract definitions and the
  promises they divide between them.
- [Charity — BNB Chain](charity-bnb.md) — how the two write
  authorities are enforced by the contract rather than by policy.
- [Discovery](discovery.md) — where assigned roles become findable.
