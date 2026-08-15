# Charity

Charitable aid runs on trust, and trust is exactly what the people
involved can't afford to spend. A donor wants to know their money
reached a real, verified program — but "trust us" is all most programs
can offer. A beneficiary needs help *now* — but asking for it usually
means handing their identity, their documents, and their situation to
strangers forever. The charity seat exists so that neither has to.

Here's the whole idea in one paragraph: charity is an **open
application seat composed of separately replaceable roles** — nobody's
word is taken for anything another party can attest to instead. The
charity operator runs the program; a credential issuer vouches for the
organization under a named policy; a financial provider moves the
money; a notary makes narrow claims verifiable; an auditor signs
reports. The user's application pins exactly which of each it accepts,
shows every claim with its author, and asks the user to authorize
exact canonical terms — never a provider's bytes. A beneficiary can
prove they are *eligible* without publishing who they *are*. And the
whole contract surface is built to carry **zero intentional donor or
beneficiary PII** in any public record.

**Contract:** [`charity/Charity.md`](https://github.com/onymchat/onym-system/blob/main/charity/Charity.md)
— the technology-free boundary this page summarizes — and
[`charity/UI-Charity.md`](https://github.com/onymchat/onym-system/blob/main/charity/UI-Charity.md),
the messenger-side application profile. Both are drafts.
**Implementations:** [Stellar/Soroban](charity-stellar.md) ·
[BNB Chain](charity-bnb.md) — both are **plans**; no charity code
exists in any Onym repository yet.

This page stays deliberately free of any one blockchain, payment rail,
or aid program — so does the contract, explicitly: it requires no
ledger, no token, no particular zero-knowledge system. The seat is one
merged boundary with two user-facing journeys, and this book presents
it through them: **Donate** and **Ask for Aid**.

## The roles, kept apart on purpose

The contract's first decision is that these are *separately
replaceable* parties, and their identities must be **separately
inspectable** — a brand, domain, or app-store listing binds none of
them to another:

| Role | What it does — and only that |
|---|---|
| **User application** | Presents canonical intent, holds local keys and capabilities, verifies receipts. Never silently decides an organization is trustworthy. |
| **Charity operator** | Defines the program, allocates aid, handles complaints. Makes the claims attached to a campaign — and owns their truthfulness. |
| **Organization credential issuer** | Checks an organization under a named policy; issues and revokes a signed credential. |
| **Eligibility issuer** | Attests that a beneficiary satisfies a named aid policy — preferably without publishing who they are. |
| **Financial provider** | Quotes, settles, refunds, disburses, under its own legal authority. |
| **Notary** | Records narrow public state and produces verifiable evidence. Nothing more. |
| **Auditor / report issuer** | Signs claims about receipts, allocation, expenditure, impact — each attributed, none inherited. |

One party may hold several roles, but every signed record still says
which authority it exercises. The messenger publisher is not implicitly
the operator, custodian, or auditor; the operator is not implicitly
allowed to administer the messenger or read its user graph.

## "Verified" is always qualified

The contract's central trust rule, worth quoting nearly verbatim:
`verified organization` is never a universal state. It means *issuer I
attested at time T that subject S satisfied policy P, version V, for
scope C, until expiry or revocation* — and the UI must show the
issuer, policy, scope, validity, and current revocation status, never
collapsing them into an unqualified checkmark. A user's `TrustPolicy`
pins which issuers and assurance levels they accept, and issuer trust
is never transitive.

Cryptographic evidence is equally narrow: a signature proves control
of a key over exact bytes; a credential proves an issuer made a claim;
a zero-knowledge proof proves its declared predicate; a settlement
receipt proves a financial transition. **None of them alone proves
absence of fraud, delivery of aid, or charitable impact** — and no
conforming UI may pretend otherwise.

## Journey one: Donate

A donor finds a **campaign** — a signed, revisioned record naming its
operator, organization, credential, purpose, accepted assets, exact
financial destinations, and its allocation, refund, and reporting
policies. Campaigns live in one state machine:

```text
draft → active → paused → active
            \-→ closed
            \-→ revoked
```

Before any money moves, the application fetches a fresh signed
**Donation Quote** — gross amount, every fee with its recipient, net
amount, destination, finality rule, refund rule, expiry — and shows
one canonical confirmation. Only then does the user authorize a
**Donation Intent** pinning the exact quote, campaign revision, and
credential digest. The donation's life:

```text
prepared → submitted → pending → finalized
finalized → refund-pending → refunded | refund-denied
finalized → reversed
```

with `expired` and `failed` exits along the way. `submitted` is not
settlement; `finalized` means only that the pinned financial profile's
finality rule was satisfied. A finalized **Donation Receipt** binds
the digest of the exact intent bytes the donor authorized, and refunds
run under the policy pinned by the *original* intent — never the
campaign's newest one.

What the donor is promised:

- **Terms cannot mutate under your signature.** A changed amount, fee,
  destination, campaign revision, or privacy choice invalidates the
  preview and requires a new decision.
- **No undisclosed cut, ever.** Any fee or revenue share deducted from
  your value is a named line — amount, rate, recipient, basis — in the
  signed quote and the finalized receipt.
- **Reporting is aggregate and attributed.** Fund-flow reports carry
  their issuer's signature, state gross versus net explicitly, exclude
  duplicates, refunds, and reversals — and never contain donor or
  beneficiary identities. Fund flow, allocation, expenditure, and
  impact are four separate claims with four separate authors.
- **Privacy is stated, not assumed.** A public rail can expose
  addresses, amounts, and timing even when the messenger carries no
  PII; the profile declares it and the UI warns before signing.
  "Anonymous" is never shown when the selected rail makes you
  identifiable.

## Journey two: Ask for Aid

A beneficiary's journey is built backwards from one requirement: prove
*eligibility*, not *identity*.

A campaign's **Eligibility Policy** names a predicate ("eligible under
policy P"), the issuers whose credentials count, the proof system, the
public inputs, and a **nullifier scope**. The beneficiary's device
builds an **Eligibility Presentation** locally — a proof of the
predicate that, where the proof profile supports it, never publishes
the underlying credential. The nullifier is **campaign- and
epoch-scoped**: it stops the same entitlement being claimed twice in
the same window, and it must never become a cross-campaign or
permanent person identifier.

An accepted presentation backs an **Aid Claim**, whose delivery
details — payout address, pickup capability, shipping coordinate —
are sealed to the named delivery provider alone. Public state may
carry only the claim digest, the scoped nullifier, and a randomized,
claim-scoped recipient commitment. The claim's life:

```text
prepared → submitted → eligibility-verified → approved → disbursed
```

with `rejected` and `expired` exits at each review step. Approval is
not disbursement, and the **Disbursement Receipt** records outcome,
entitlement, fees, and evidence without repeating private delivery
details.

What the beneficiary is promised:

- **No roster.** A beneficiary list is never published — not by the
  operator, not by the notary, not in a report.
- **Failure is private.** A failed proof or a duplicate-claim refusal
  (`NULLIFIER_USED`) surfaces as a typed, scoped outcome — never a
  public person identifier, never leaked diagnostics.
- **Case files stay with the responsible party.** Source credentials
  and documents remain with the issuer or operator under declared
  retention — never in public notary state, never shown to donors.
- **Safe exit.** A beneficiary can abandon the flow without sending
  partial proofs, and no notification or screenshot surface reveals
  aid participation by default.

## The privacy boundary

The base interface requires no real name, email, phone, device
fingerprint, address-book upload, or referral token from anyone. A
provider may lawfully need regulated data for a specific rail or
jurisdiction — but that is a separate, explicit handoff with a named
controller, purpose, and retention, never a silent default. No
provider may ever demand the identity root secret or an unscoped key.

The honest formulation — the contract refuses to promise more: the
target is **zero intentional PII publication** in the messenger
protocol objects and the charity contract surface, backed by negative
fixtures. That is a testable design invariant, not a warranty that no
endpoint can ever be compromised, and each deployment declares who
controls which data set, for what purpose, and who answers when
something goes wrong.

Measurement follows the same rule: fund flow is measured, people are
not. `eligibleDonationVolume` counts finalized, non-refunded receipts
into pinned destinations — it proves qualifying financial flow and
deliberately cannot prove that any particular person saw a message,
installed an app, or was "converted."

## Where the code is

Nowhere, yet — and this book doesn't pretend otherwise. The abstract
boundary leaves ledgers, proof systems, and rails to implementation
profiles, and two are planned:

- **[Stellar/Soroban](charity-stellar.md)** — the planned reference
  binding: Soroban contracts as the campaign's notary and (later)
  financial bindings, riding the notary seat's running relayer and
  manifest infrastructure.
- **[BNB Chain](charity-bnb.md)** — the planned EVM sibling: Solidity
  contracts with toolchain-generated PLONK **BN254** verifiers,
  following the [notary's BNB plan](notary-bnb.md).

## Next steps

- [Notary](notary.md) — the seat a charity deployment binds to for
  narrow public state, and whose operator-manifest discipline the
  implementations reuse.
- [Discovery](discovery.md) — where charity deployments and their
  provider bindings will be listed once they exist.
