# Charity — BNB Chain

*Seat implementation page, draft 0.3 — 15 August 2026. Specification:
merged. Code: none — see [Honest status](#honest-status).*

**Profile:** [`charity/UI-Charity-BNB.md`](https://github.com/onymchat/onym-system/blob/main/charity/UI-Charity-BNB.md)
— merged in `onym-system` (draft 0.1). It binds the abstract
[`charity/Charity.md`](https://github.com/onymchat/onym-system/blob/main/charity/Charity.md)
boundary's **notary and eligibility bindings** to BNB Smart Chain
(mainnet 56, testnet 97; opBNB explicitly out of scope). An EVM
sibling of the [Stellar plan](charity-stellar.md), not its
replacement.
**Code:** none.

This page summarizes what the profile specifies, so a reader can judge
the design without the notary background. The full normative text —
every signature, error, encoding rule, fixture, and open question — is
the profile itself.

## Where the notary boundary ends

The contract (`cha-anchor`, one non-upgradeable instance per charity
deployment) does exactly what `UI-Charity.md` §8.1 scopes the notary
port to: campaign status and revision commitments, donation and
disbursement **receipt commitments**, fund-flow commitments,
**nullifier uniqueness** per campaign and epoch, and inspectable
policy/status changes.

It holds no funds, mints nothing, and transfers nothing. The profile
prohibits payable entrypoints, escrow, and disbursement logic under
its profile ID. An anchored receipt digest proves *the operator
anchored those exact bytes at that time* — never that money moved or
that aid arrived. Custody, settlement, refunds, and disbursement stay
with the financial provider under its own legal authority; a future
on-chain settlement rail would be a separate `financialBindings`
profile with its own finality, refund, and reversal mapping, which the
profile deliberately does not define.

## Two write authorities, contract-enforced

The interface splits into two classes the contract keeps distinct:

- **Operator-attested writes** — campaign registration, revision
  advance (by exactly one, no skips or rewinds), status changes
  mirroring the abstract campaign machine
  (`active/paused/closed/revoked`, no exit from closed or revoked —
  registration itself is the machine's draft→active edge, drafts stay
  off-chain), policy registration, and receipt/report anchors. Gated
  on an operator-admin address baked immutably into the bytecode.
  These are tamper-evident operator *statements*: the chain proves who
  said it and when, not that it is true. The status gate scopes to
  *new authorizations* only — receipt, disbursement, and report
  anchors record already-authorized operations and stay writable after
  a campaign pauses, closes, or is revoked, so ending a campaign can
  never truncate its own audit trail.
- **Proof-authorized writes** — `anchorAidClaim`, sender-agnostic.
  Anyone may submit it; validity comes from a PLONK/BN254 eligibility
  proof checked against the contract's own state, never from
  `msg.sender`. One transaction verifies the proof, checks campaign
  status, revision, and epoch, **consumes the nullifier, and anchors
  the claim atomically** — there is no separate verify step that could
  open a window between eligibility check and nullifier consumption.

The profile requires a fixture that checks the split both ways:
operator entrypoints refuse other senders, and claim anchoring
succeeds from an arbitrary account. (No fixture exists yet — see
[Honest status](#honest-status); "requires" is the strongest true
verb on this page.)

The public trail is the contract's typed events — `CampaignRegistered`
and `CampaignRevisionAdvanced`, `CampaignStatusChanged`,
`DonationReceiptAnchored`, `AidClaimAnchored`, and
`DisbursementAnchored` — keyed by campaign and, for the claim
lifecycle, by `claimDigest`, the public join between a claim and its
disbursement. These are the names the
[adversary page](charity-adversary.md) analyzes; they are defined by
the merged profile and would change only with it.

## Errors carry their own retry semantics

Every custom error is a distinct four-byte selector with a normative
retry class, because "what should the client do now" differs:

| Class | Errors | Client behavior |
|---|---|---|
| refresh-and-rebuild | `StaleCampaignRevision`, `EpochNotCurrent` | Re-resolve, re-consent to the new revision, rebuild the presentation (a new epoch also derives a new nullifier), resubmit. |
| terminal scoped refusal | `NullifierUsed` | The entitlement was already claimed in this campaign and epoch. Show the scoped refusal; never a person identifier, never a retry. |
| refuse-as-defect | `InvalidProof`, `ValueNotInField`, `UnknownPolicy`, `CampaignExists`, `OperatorOnly`, … | Generator or tooling bug; retrying cannot help. Proof diagnostics stay private. |
| security event | `AnchorConflict`, post-inclusion reorg contradiction | Preserve evidence, raise the incident path, never silently resubmit or overwrite. |

Because the contract **derives its own public-input vector** from call
arguments and storage, the notary profile's "misordered statement"
failure cannot arise here by construction — so there is deliberately
no `PublicInputsMismatch` selector; each bindable field has its own
named check, decided before the expensive pairing check runs. (An
earlier revision of this page borrowed the notary's error names
verbatim; the profile refines them.)

## Nullifiers, commitments, and encodings

The design choices a skeptical reader should check, in brief:

- **Nullifier** = Poseidon(tag, credentialSecret, campaignId,
  epochIndex), computed in-circuit and constrained to the same secret
  that satisfies the predicate. Campaign revision is deliberately *not*
  an input, so a policy update cannot mint a second claim in the same
  window — the profile names a fixture for exactly that. Cross-campaign
  and cross-epoch unlinkability rests on the hash assumption, and the
  profile says so rather than claiming a fixture proves it.
- **Recipient commitment** = keccak(tag ‖ delivery-binding digest ‖
  fresh randomness), binding-only, opened privately to the delivery
  provider. It hides the delivery coordinate and whether two claims
  share one; it does not hide that a claim exists, its timing, or the
  public claim→disbursement join, which is the audit trail by design.
- **Encodings**: freely chosen identifiers are drawn in-field by
  rejection sampling and rejected on-chain when out of range; digests
  enter the circuit as two 128-bit limbs (injective, no grinding);
  **modular reduction is forbidden everywhere**, with the aliasing
  attack it would enable spelled out in the profile. Three ASCII
  domain-separation tags are fixed, and a statement-tag constant in
  the circuit makes a valid proof for any *other* statement family —
  including the notary's group-transition circuits — unverifiable
  here.

## Why BN254, in charity terms

The eligibility verifier is the only cryptographic novelty this
binding adds, and it is the one component a charity deployment can
least afford to get wrong: it decides whether an unnamed person's
claim on real aid is honored. The profile therefore chooses standard
PLONK over BN254 with **toolchain-generated** Solidity verifiers: the
BN254 pairing precompiles have years of production exposure, verifier
generation is automated and audited in widely used toolchains, and no
hand-written verifier needs a bespoke audit. The cost is a real one —
BN254 circuits, setup, and verifying keys separate from any future
BLS12-381 Stellar sibling, and a cross-curve proof is invalid evidence
in both directions, with fixtures required for both rejections — one
of which necessarily ships as a published vector with a
counterpart-unimplemented marker until a Stellar sibling exists (see
[Build order](#build-order)).

Deployment identity is chain ID + contract address + runtime code
hash, all three verified before first use; the verifier and admin
addresses are `immutable` values inside that hashed bytecode; proxies
and `delegatecall` dispatch are prohibited under this profile ID.

## The operator manifest

The relayer's signed, byte-served operator manifest — live today on
the Stellar notary side — would gain the charity profile entry, the
`cha-anchor` deployments it administers, and `eip155` network entries
binding the ed25519 operator identity to the secp256k1
`submitterAccount` that pays gas and the `adminAccount` whose
`msg.sender` the operator-attested entrypoints accept. The normative
client check carries over from the
[notary BNB profile](notary-bnb.md#the-operator-manifest-already-exists)
unchanged: deployment verification MUST compare the manifest's
`adminAccount` against the contract's `getOperatorAdmin()` (and the
declared verifier against `getVerifier()`) — without that comparison,
"declared powers match contract-enforced reality" is not verifiable.
A deployment absent from the manifest does not exist for clients,
whatever is on the chain.

## Receipts, finality, and reorgs

Every write returns the EVM transaction hash from the first release —
mandatory, but *provisional*, because a same-nonce fee-bump
replacement changes the hash without changing the operation. The
stable reconciliation key is the client's operation ID, with an
outcome query returning current and superseded hashes. Receipt
(`status == 1`) and the BSC `finalized` checkpoint are distinct
client-visible states, and nothing is shown as final before
reconciliation against the finalized tag. A receipt in a
later-orphaned block is a `conflicting_state` **security event, not a
retry** — for a claim write, the rebuild path terminates in either an
idempotent identical anchor or `NullifierUsed`, both correct, and the
event is still reported.

## What a conforming implementation must refuse

The profile's fixture catalogue is weighted toward negatives on
purpose — the binding's value is what it refuses. Among them: a second
claim from the same credential in the same scope; the same claim after
a campaign-revision advance (nullifier stability); a BLS12-381 proof
under this profile and a BN254 proof under the Stellar one; a valid
BN254 proof for a *different* statement family; out-of-field and
reduction-aliased identifiers; stale revisions; conflicting anchors
under one key; and a three-layer PII fixture that plants names, IBANs,
emails, and addresses in the input objects and greps every emitted log
and written storage slot for them — zero hits to pass, with sealed
recipient payloads asserted absent from calldata entirely. The full
named list, precise enough to implement from, is
[profile §13](https://github.com/onymchat/onym-system/blob/main/charity/UI-Charity-BNB.md#13-conformance-fixtures).

For what a block-explorer adversary can still see and infer — anchor
counts, timing, the single gas-paying submitter — and what the UI must
disclose before anyone signs, see the
[adversary's view](charity-adversary.md).

## Honest status

- **Nothing on this page runs.** No BN254 charity circuits, no
  Solidity, no EVM charity endpoints in the relayer, no fixtures. The
  [profile](https://github.com/onymchat/onym-system/blob/main/charity/UI-Charity-BNB.md)
  is merged — a specification, not code.
- **Two shared unbuilt dependencies remain, both built once and
  reused:** the relayer's **EVM backend** and the mobile Rust FFI's
  **BN254 prover backend**. Both this binding and the
  [notary EVM plan](notary-bnb.md) need each, neither build has
  either, and whichever build reaches one first implements it for
  both — the profile's §15 says "built once, not twice" in exactly
  those terms. Delivery is *not* sequenced after the
  [Stellar charity build](charity-stellar.md), which has no code and
  no profile of its own yet.
- The abstract contracts it answers to (`Charity.md`,
  `UI-Charity.md`) are merged drafts (0.1, August 2026); the profile
  flags one wording question in `Charity.md` §6.8 (campaign-scoped
  fields in public claim anchors) for upstream decision rather than
  assuming an answer.

## Build order

BNB proceeds on its own dependency chain. The
[Stellar plan](charity-stellar.md) is a sibling, not a gate — neither
binding waits for the other:

1. **Profile merged** —
   [`charity/UI-Charity-BNB.md`](https://github.com/onymchat/onym-system/blob/main/charity/UI-Charity-BNB.md)
   landed in `onym-system`. Done; everything below is not.
2. **Circuits** — the `membership-set-v1` BN254 constraint system,
   setup, and verifying keys, publishing their logical vectors as they
   land. The cross-curve cross-check binds whichever sibling profile
   arrives *second*, in either direction.
3. **Prover** — the BN254 PLONK backend in the mobile Rust FFI, built
   once and shared with the [notary EVM plan](notary-bnb.md) exactly
   like the relayer backend below; without it, no client can generate
   the proof bytes the conformance chain starts from.
4. **Contracts** — `cha-anchor` plus generated verifiers, deployed
   immutably.
5. **Relayer EVM backend** — built once and shared with the notary
   EVM plan: whichever build reaches it first implements it, the
   other reuses it. Charity must not fork its own EVM plumbing — and
   must not wait for the notary build if charity reaches it first.
6. **Declare, list, prove** — manifest entries, discovery listing, and
   the fixture suite green, in that order, before any real campaign
   binds this deployment.

What independence costs, stated rather than hidden: until a Stellar
sibling exists, the cross-curve rejection fixture ships one-directional
(with the counterpart-unimplemented marker the profile specifies), and
the proved-against-real-campaigns assurance must come from this
binding's own testnet pilot.

## Next steps

- [Charity](charity.md) — the abstract seat this binds.
- [The adversary's view](charity-adversary.md) — what a public EVM
  rail exposes, mitigation by mitigation.
- [Who holds which role](charity-roles.md) — the abstract roles mapped
  to concrete parties, including the unassigned ones.
- [Notary — BNB Chain](notary-bnb.md) — the merged EVM profile whose
  hardening rules the charity profile restates.
