# Charity on a public chain: the adversary's view

*Threat-model page, draft 0.2 — 15 August 2026. **Nothing analyzed
here runs**: no contract, no circuits, no fixtures — see the
[BNB binding's honest status](charity-bnb.md#honest-status). The
observer below inspects the binding as specified; every "what they
see" is what they *would* see on a deployment that does not exist.
The [Stellar plan](charity-stellar.md) shares most rows with a
smaller explorer ecosystem. The
[Cardano plan](charity-cardano.md#every-surface-that-can-carry-bytes)
states four deltas — decoded inline datums, per-campaign
nullifier-token pages, a persistent collateral pool, and no event log
at all — and they stay on that page rather than becoming rows here
because they follow from a ledger model this page's row structure
assumes away. The
[Solana plan](charity-solana.md#every-surface-that-can-carry-bytes)
states three of its own — indexed program logs, a first-class fee
payer on every claim, and, if that binding takes its proposed
on-chain settlement, a follow-the-money view no sibling offers — and
its
[settlement section](charity-solana.md#settlement-is-where-this-binding-gets-hard)
is a threat model this page's structure does not yet cover, because
no other binding puts a payout address on the ledger at all. Both
move here when those bindings have profiles to analyze; the
inferences below are the ones that hold across all four.*

The charity trail puts commitments, nullifiers, and status changes on
a public, heavily indexed chain on purpose — that is what makes the
operator's procedure independently checkable. The honest question is
what that transparency costs, and the abstract contract requires the
answer be *declared*, not discovered: `Charity.md` §9.3 obliges every
profile to state its public-rail exposure, and §9.4 refuses the
comforting lie of "zero incidents" in favor of "zero intentional PII
publication" as a testable invariant.

So assume the strongest realistic observer: a block explorer, the
contract ABI, this documentation, and **off-chain knowledge** — they
read the community's chat, they know who mentioned applying for aid,
they watch the operator's public reporting. For each thing they can
do, this page states the mitigation, the residual risk that remains
*after* the mitigation, and what the UI must disclose before a user
signs. Where a mitigation is undecided or unproven, it says so; a
mitigation this page does not list is a mitigation that does not
exist.

## What the observer never gets

Grounding first, because the rest of the page is about inference, not
disclosure. Under the profile's PII fixture, the chain never carries:
names, contact details, addresses, documents, bank or payout
coordinates, credential contents, per-donation amounts (this binding
anchors no amounts — money moves off-rail and only receipt *digests*
land on-chain), or the sealed recipient payload, which is excluded
from calldata entirely, not merely encrypted within it. Everything
below is what an observer can *infer around* those exclusions.

## Inference by inference

### 1. Anchor counts and timing

**What they see.** Every `AidClaimAnchored` and
`DonationReceiptAnchored` event, timestamped by consensus, per
campaign, forever, trivially chartable by any explorer.

**What they can infer.** Campaign activity levels; claim cadence per
epoch; and — the sharp edge — **small-count correlation**: a campaign
serving a small community that anchors three claims in a week, read
against off-chain knowledge of who mentioned applying, can narrow
"which pseudonymous claims are whose" to a small candidate set.

**Mitigation.** Partial. The contract carries no identities to
confirm a guess against, and `Charity.md` §6.9 requires small counts
in *reports* to be bucketed or suppressed. But §6.9 does not cover
anchors, and whether the operator should batch anchors on a declared
schedule — and at what minimum batch — is an explicitly open question
in the
[merged profile](https://github.com/onymchat/onym-system/blob/main/charity/UI-Charity-BNB.md#14-open-questions)
(§14.5), with a written analysis required before
batching may be *claimed* as a mitigation. Until then: **no timing
mitigation exists at the anchor layer**.

**Residual risk.** Real for small communities. Membership of the
candidate set is inferable; confirmation still requires off-chain
information the chain does not hold.

**UI must disclose, before a claimant signs:** that the existence,
campaign, epoch, and timing of the claim are permanently public, and
that in a small community this may narrow who claimed to those who
watch closely.

### 2. Nullifier activity

**What they see.** Every spent nullifier, keyed by campaign and
epoch.

**What they can infer.** How many distinct entitlements were claimed
per scope — by design, that is the duplicate-prevention audit. Within
one scope, nothing links a nullifier to a person without the
credential secret.

**Mitigation.** Scoped derivation: campaign and epoch are hashed into
the nullifier with the credential secret, so cross-campaign and
cross-epoch values are unlinkable **under the hash's pseudorandomness
assumption and a correctly constrained circuit**. The profile's
specified fixtures — none of which exist yet — would prove only that
the derivation is scoped; no fixture can prove unlinkability itself,
and the profile states this limit rather than papering over it.

**Residual risk.** A circuit soundness bug or hash break would
degrade unlinkability; and *counts* per scope remain public (feeding
inference 1).

**UI must disclose:** that a per-campaign, per-epoch claim marker is
public, and that it is designed — not merely hoped — to be unlinkable
across campaigns.

### 3. The claim→disbursement join

**What they see.** `DisbursementAnchored` sharing the `claimDigest`
key with its `AidClaimAnchored`.

**What they can infer.** That a specific pseudonymous claim was (or
was not, or was only later) disbursed, and the delay between the two.

**Mitigation.** None, deliberately: the join *is* the audit trail —
it is what lets anyone verify that disbursements trace to verified
claims. The timing side-channel it creates is inference 4's row.

**Residual risk.** Disbursement timing is public per claim; combined
with inference 1's candidate sets, an observer may estimate when a
particular person was paid.

**UI must disclose:** that claim outcomes and their timing are
public at the pseudonym level.

### 4. On-chain/off-chain timing correlation

**What they see.** A disbursement anchor at time T on-chain; and, off
chain, whatever they can observe of real-world payouts — a bank
transfer a recipient mentions, an operator's public report, a
community thank-you post.

**What they can infer.** Joins between the pseudonymous trail and
real-world events, at whatever precision the two timestamps allow.

**Mitigation.** Undecided, and therefore **not claimed**: anchoring
on a declared schedule rather than immediately after payout would
coarsen the correlation, but it is the same open question as batching
(profile §14.5) and must not be promised before it is analyzed and
specified. What *is* specified: the anchor carries no amount and no
rail reference, so the join gains a time, not a sum or an account.

**Residual risk.** An observer with good off-chain visibility into a
small community can probably join some anchors to some events. The
trail is pseudonymous, not covert.

**UI must disclose:** that anchoring follows real-world events
closely enough that an observer with off-chain knowledge may
correlate them.

### 5. The single gas-paying submitter

**What they see.** One relayer EOA signs and pays per EVM deployment
it serves — and the operator's signed manifest publicly binds that
EOA to the same Ed25519 operator identity as its other-chain
deployments (today, the Stellar notary submitter). The linkage across
chains is the manifest, by design; the linkage within a chain is the
EOA itself. Either way, one inspectable operational graph.

**What they can infer.** Which deployments share an operator; the
operator's full activity rhythm; and that a claim was submitted
*through that operator* (though `anchorAidClaim` is sender-agnostic,
so any funded account could submit — a claimant using their own EOA
would deanonymize themselves against inference 2's protections, and
the UI must never suggest it as a privacy improvement).

**Mitigation.** None within the shared-operator model — this is an
accepted, *declared* property: the operator's manifest privacy
profile must name it, exactly as the notary EVM profile requires. The
alternative (user-paid gas) is worse for the user and is excluded for
the same linkability reasons the notary profile excludes it.

**Residual risk.** Operator-level metadata concentration: the relayer
observes IP, timing, and payload sizes for everything it submits, and
its EOA broadcasts its aggregate activity.

**UI must disclose:** which operator submits, what that operator can
observe (per its signed privacy profile), and that all its
deployments are publicly linkable through its submitting account.

### 6. Campaign and operator surveillance

**What they see.** Campaign registrations, revision advances, pauses,
closures, revocations, policy registrations — the operator's whole
administrative history under its admin address, plus the contract
address itself, which explorers will label.

**What they can infer.** That an organization runs an aid program,
when it changed terms, when it paused. For an operator serving
at-risk communities, *the program's existence and rhythm* is itself
sensitive.

**Mitigation.** None on-chain; running a public-trail charity is a
public act, and the seat is honest that the operator is the
accountable, visible party. Campaign *content* (title, purpose,
eligibility text) is off-chain behind a digest — the chain shows that
a campaign exists and changed, not what it says; who may resolve the
content is the deployment's choice, not the chain's.

**Residual risk.** A hostile observer learns the program's tempo and
scale. Operators for whom that is unacceptable should not bind a
public-chain trail, and the seat page's honest answer is that this
binding is the wrong tool for them.

**UI must disclose** (operator-facing, at deployment creation): that
the administrative history of the deployment is permanently public
under its admin address.

### 7. Reorganizations and rewrites

**What they see / do.** A stronger adversary participates: attempts
to orphan blocks carrying anchors, or to front-run a claim submission.

**What it buys them.** Little, by construction. A reorged anchor is a
`conflicting_state` security event — reported, never silently
retried — and the rebuild terminates in an idempotent identical
anchor or `NullifierUsed`. Front-running a claim with copied calldata
anchors the *same* claim (same digest, same nullifier, same
commitment — the proof binds all of them); it cannot redirect a claim
or mint a different one. Post-finality rewrites are outside this
profile's adversary budget and inherit BSC's fast-finality
assumptions, which the binding pins explicitly.

**Residual risk.** Denial and delay around the finality window;
detection is specified, prevention is the chain's consensus.

**UI must disclose:** nothing additional per-signature; the
finality rule itself is part of the pre-signature terms the abstract
contract already requires.

## Reading this page honestly

Five of the seven rows offer no *effective* mitigation at their
layer: rows 3, 5,
and 6 say **none** outright — two of them deliberately (the
claim→disbursement join *is* the audit trail; the operator's public
history is the accountability), one as an accepted, declared property
of the shared-submitter model — while row 1 is only partial and row
4's single candidate mitigation (batching) is explicitly not yet
earned. Only row 2's scoped derivation and row 7's
detection-plus-finality are designed mitigations, and row 2's rests
on a stated assumption. That is the intended reading: the
binding's privacy claim is *zero intentional PII publication plus
stated inference surface* — not anonymity. A deployment whose threat
model requires the anchors themselves to be covert needs a different
binding, and no page in this book should talk it out of that
conclusion.
