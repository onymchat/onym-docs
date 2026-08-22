# Charity — BNB Chain

*Seat implementation page, draft 0.3 — 15 August 2026.*

**Status:** Specification merged; code not implemented. See
[Honest status](#honest-status).

**Profile:** [`charity/UI-Charity-BNB.md`](https://github.com/onymchat/onym-system/blob/main/charity/UI-Charity-BNB.md),
merged as draft 0.1. It binds the
[`charity/Charity.md`](https://github.com/onymchat/onym-system/blob/main/charity/Charity.md)
notary/eligibility boundaries to BNB Smart Chain. It covers mainnet 56 and
testnet 97, not opBNB. An EVM sibling of the
[Stellar plan](charity-stellar.md), not its replacement. **Code:** none.

The profile is normative for signatures, errors, encodings, fixtures, and open
questions.

## Where the notary boundary ends

Each deployment has one non-upgradeable `cha-anchor`. It handles only the
`UI-Charity.md` §8.1 notary port: campaign status and revision
commitments, donation and disbursement **receipt commitments**, fund-flow
commitments, **nullifier uniqueness** per campaign and epoch, and inspectable
policy and status changes.

It holds no funds, mints nothing, and transfers nothing. Its profile ID
prohibits payable entrypoints, escrow, and disbursement logic. An anchored
digest proves *the operator anchored those exact bytes at that time*, never
that money moved or aid arrived.

The financial provider retains custody, settlement, refunds, and disbursement
under its legal authority. Any on-chain rail needs a separate
`financialBindings` profile, finality, refund, and reversal mapping.

## Two write authorities, contract-enforced

- **Operator-attested writes:** campaign/policy registration; receipt/report
  anchors; revision advances of exactly one, without skips or rewinds;
  and `active/paused/closed/revoked` changes. Registration is the draft→active
  edge. Drafts stay off-chain; closed and revoked states have no exit. An
  immutable admin gates them. The chain proves author and time, not truth.
  Status gates only new authorizations. Receipt,
  disbursement, and report anchors remain writable after pause, closure, or
  revocation, preserving the audit trail.
- **Proof-authorized writes:** `anchorAidClaim` is sender-agnostic. Anyone may
  submit it. A PLONK/BN254 eligibility proof against contract state supplies
  validity, never `msg.sender`. One transaction checks status, revision, and
  epoch, verifies the proof, **consumes the nullifier, and anchors the claim
  atomically**. There is no separate verification window.

A required fixture makes operator entrypoints reject other senders while an
arbitrary account anchors a claim. None exists; “requires” is the strongest
true verb. See [Honest status](#honest-status).

Typed events form the public trail: `CampaignRegistered`,
`CampaignRevisionAdvanced`, `CampaignStatusChanged`,
`DonationReceiptAnchored`, `AidClaimAnchored`, and `DisbursementAnchored`.
Campaign keys index them; `claimDigest` is the public claim-to-disbursement
join. The [adversary page](charity-adversary.md) analyses these profile-defined
names. Only the profile can change them.

## Errors carry their own retry semantics

Every custom error has a distinct four-byte selector and normative retry class.

| Class | Errors | Client behaviour |
|---|---|---|
| refresh-and-rebuild | `StaleCampaignRevision`, `EpochNotCurrent` | Re-resolve, re-consent to the new revision, rebuild the presentation, and resubmit. A new epoch derives a new nullifier |
| terminal scoped refusal | `NullifierUsed` | Already claimed in this campaign and epoch. Show the scoped refusal; never a person identifier or retry |
| refuse-as-defect | `InvalidProof`, `ValueNotInField`, `UnknownPolicy`, `CampaignExists`, `OperatorOnly`, … | Generator or tooling bug; retry cannot help. Keep diagnostics private |
| security event | `AnchorConflict`, post-inclusion reorg contradiction | Preserve evidence, raise the incident, and never silently resubmit or overwrite |

The contract derives public inputs from arguments and storage. Misordering
cannot occur, so no `PublicInputsMismatch` selector exists. Each field has a
named pre-pairing check. Earlier borrowed notary errors are replaced.

## Nullifiers, commitments, and encodings

- **Nullifier:** Poseidon(tag, credentialSecret, campaignId, epochIndex), tied
  in-circuit to the secret satisfying the predicate. Revision is excluded, so
  policy changes cannot create a second same-window claim. A fixture covers
  that. Cross-campaign and cross-epoch unlinkability rests on the hash
  assumption, not a fixture.
- **Recipient commitment:** keccak(tag ‖ delivery-binding digest ‖ fresh
  randomness), opened privately to the delivery provider. It hides the delivery
  coordinate and reuse between two claims, but not claim existence, timing, or
  the public claim-to-disbursement join.
- **Encodings:** rejection sampling draws freely chosen identifiers in-field;
  out-of-range values are rejected. Digests use two injective 128-bit limbs
  without grinding. **Modular reduction is forbidden everywhere** because it
  enables aliasing. Three fixed ASCII tags separate domains. A statement-tag
  constant rejects valid proofs for other families, including notary
  group-transition circuits.

## Why BN254, in charity terms

The profile chooses standard PLONK over BN254 with **toolchain-generated**
Solidity verifiers. BN254 pairing precompiles have years of production exposure.
Widely used toolchains automate audited generation, avoiding bespoke review of
handwritten code. The verifier decides an unnamed person's claim on real aid
and is the only cryptographic novelty.

BN254 circuits, setup, and keys remain separate from a future BLS12-381
Stellar sibling. Cross-curve proofs fail in both directions; fixtures must test
both. Until Stellar exists, one ships as a published vector with a
counterpart-unimplemented marker. See [Build order](#build-order).

Before first use, clients verify all three: chain ID + contract address + runtime
code hash.
Verifier/admin addresses are `immutable` in that bytecode. Proxies and
`delegatecall` dispatch are prohibited under this profile ID.

## The operator manifest

The live, signed, byte-served Stellar notary manifest would add the charity
profile, `cha-anchor` deployments, and `eip155` entries. They bind the Ed25519
operator identity to the gas-paying secp256k1 `submitterAccount` and the
`adminAccount` accepted as `msg.sender`.

Under the [notary BNB profile](notary-bnb.md#the-operator-manifest-already-exists),
verification MUST compare `adminAccount` with `getOperatorAdmin()` and the
declared verifier with `getVerifier()`. Otherwise declared powers cannot match
contract-enforced reality. A deployment absent from the manifest does not exist
for clients, regardless of chain state.

## Receipts, finality, and reorgs

Every first-release write returns a mandatory but *provisional* EVM transaction
hash. Same-nonce fee-bump replacement changes the hash, not the operation. The
client operation ID is stable; outcome queries return current/superseded
hashes.

Receipt (`status == 1`) and BSC `finalized` are distinct client states. Nothing
is final before reconciliation against the finalized tag. A later-orphaned
receipt is a `conflicting_state` **security event, not a retry**. Rebuilding
yields an identical idempotent anchor or `NullifierUsed`. Both are correct; the
event remains reportable.

## What a conforming implementation must refuse

- a second claim from the same credential and scope;
- the same claim after campaign revision advances, proving nullifier stability;
- a BLS12-381 proof here and a BN254 proof under the Stellar profile;
- a valid BN254 proof for another statement family;
- out-of-field and reduction-aliased identifiers;
- stale revisions and conflicting anchors under one key; and
- a three-layer PII test planting names, IBANs, emails, and addresses in input
  objects, then grepping every emitted log and written storage slot.

Passing requires zero PII hits and no sealed recipient payload in calldata.
[Profile §13](https://github.com/onymchat/onym-system/blob/main/charity/UI-Charity-BNB.md#13-conformance-fixtures)
contains the complete implementable list.

The [adversary's view](charity-adversary.md) covers anchor counts, timing, the
single gas payer, mitigations, and required pre-signing disclosures.

## Honest status

- **Nothing runs.** There are no BN254 charity circuits, Solidity contracts,
  EVM charity relayer endpoints, or fixtures. The merged
  [profile](https://github.com/onymchat/onym-system/blob/main/charity/UI-Charity-BNB.md)
  is a specification, not code.
- **Two shared dependencies remain unbuilt:** the relayer **EVM backend** and
  mobile Rust FFI **BN254 prover backend**. This binding and the
  [notary EVM plan](notary-bnb.md) need both; neither has them. The first to
  reach one implements it for both. Profile §15 requires “built once, not
  twice”. Delivery does not wait for the [Stellar charity build](charity-stellar.md),
  which has neither code nor profile.
- `Charity.md` and `UI-Charity.md` are merged drafts, version 0.1 from August
  2026. One `Charity.md` §6.8 question remains upstream: may public claim
  anchors contain campaign-scoped fields?

## Build order

BNB and the [Stellar plan](charity-stellar.md) do not gate each other.

1. **Profile merged.**
   [`charity/UI-Charity-BNB.md`](https://github.com/onymchat/onym-system/blob/main/charity/UI-Charity-BNB.md)
   landed in `onym-system`. Done; later steps are not.
2. **Circuits.** Build the `membership-set-v1` BN254 constraints, setup, and
   keys, publishing logical vectors as they land. The second sibling profile
   supplies the cross-curve check.
3. **Prover.** Build the mobile Rust FFI BN254 PLONK backend once for this
   binding and the [notary EVM plan](notary-bnb.md). Clients need it to generate
   conformance proofs.
4. **Contracts.** Deploy `cha-anchor` and generated verifiers immutably.
5. **Relayer EVM backend.** Build it once for charity and notary. Whichever
   arrives first implements it; the other reuses it. Charity must neither fork
   EVM plumbing nor wait for notary.
6. **Declare, list, prove.** Add manifest entries, then discovery listing, then
   pass the fixture suite before binding any real campaign.

Before Stellar exists, cross-curve rejection is one-directional with the
counterpart-unimplemented marker. This binding's testnet pilot must provide
real-campaign assurance.

## Next steps

- [Charity](charity.md) — abstract seat.
- [The adversary's view](charity-adversary.md) — public EVM exposure and
  mitigations.
- [Who holds which role](charity-roles.md) — abstract roles mapped to parties,
  including unassigned ones.
- [Cardano](charity-cardano.md) — the eUTXO sibling meeting the same obligations
  without `msg.sender`, mutable mappings, or typed revert selectors.
- [Solana](charity-solana.md) — shares the **curve**, not the proof system. It
  uses BN254 Groth16 native syscalls, per-circuit setup instead of this universal
  SRS, and a separate prover.
- [Notary — BNB Chain](notary-bnb.md) — the merged EVM profile whose hardening
  rules are restated here.
