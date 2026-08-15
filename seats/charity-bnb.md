# Charity — BNB Chain (plan)

This page describes an implementation that **does not exist yet** —
and, unlike the [notary's BNB plan](notary-bnb.md), its profile
document is not merged either. Read this as a design intention that
depends on the Stellar charity plan and the notary BNB profile both
landing first.

**Profile:** must be written — a `charity/UI-Charity-BNB.md` in
`onym-system`, an EVM sibling of the
[Stellar plan](charity-stellar.md)'s profile, not its replacement.
Both bind the same abstract
[`charity/Charity.md`](https://github.com/onymchat/onym-system/blob/main/charity/Charity.md)
boundary.
**Code:** none.

## What it will be

The same charity **notary binding** — campaign revision commitments,
receipt commitments, campaign- and epoch-scoped nullifier uniqueness,
fund-flow anchors, policy-change records — as Solidity on **BNB Smart
Chain** (mainnet 56, testnet 97), with the eligibility predicates
re-proved as standard **PLONK over BN254 (KZG)** and checked by
toolchain-generated Solidity verifiers. The reasoning is inherited
wholesale from the [notary BNB profile](notary-bnb.md): BN254's
pairing precompiles have years of production exposure, verifier
generation is automated and audited in widely used toolchains, gas is
cheaper, and no hand-written verifier needs an audit. The cost is
likewise inherited: new constraint systems, a new setup, new verifying
keys, and a second prover backend — a BLS12-381 eligibility proof is
not valid evidence under this profile, and a BN254 proof is not valid
under the Stellar one. Fixtures must prove both rejections.

The EVM hardening rules from the notary profile apply unchanged, and
are restated in the charity profile rather than referenced silently:

- client-chosen 32-byte values (claim digests, nullifiers, recipient
  commitments) generated **in-field** by rejection sampling and
  rejected by the contract when out of range;
- verifier and operator-admin addresses as Solidity `immutable`
  values covered by the pinned runtime code hash;
- upgradeable proxies prohibited outright;
- distinct custom errors where retry guidance differs — a stale
  campaign revision is refresh-and-rebuild, a public-inputs mismatch
  is refuse-as-defect, and a spent nullifier is `NULLIFIER_USED`, a
  terminal scoped refusal;
- transaction hash mandatory in every write receipt from the first
  release, with the operation ID as the stable reconciliation key.

## What the EVM changes for a charity trail

The values on-chain stay exactly as opaque as on Stellar — that
invariant does not move. What moves is the *neighborhood*:

- **Indexing.** BSC calldata, state, and events sit on public,
  heavily-indexed explorers. Commitment counts, nullifier activity,
  and timing become trivially chartable by anyone. For a transparency
  trail that is mostly a feature — but the profile must declare it as
  metadata exposure, exactly as `Charity.md` §9.3 requires for any
  public rail.
- **Correlation.** Zero PII on-chain does not mean zero inference
  risk: public timing plus off-chain knowledge can correlate. The
  scoped-nullifier and randomized recipient-commitment rules exist
  precisely for this, and the profile must show its work — negative
  fixtures for cross-campaign linkage, not just for PII fields.
- **One submitter.** The relayer's single gas-paying account visibly
  links every deployment it serves. That is the operator's declared
  role, not a leak — but the deployment record must say so.
- **Finality.** Receipt (`status == 1`) and the `finalized` checkpoint
  are distinct client-visible states; a receipt in a later-orphaned
  block is a security event, not a retry.

## The operator manifest

Same pattern, same file: the live manifest at
`relayer.onym.app/manifest.json` would gain the charity BNB profile
entry and `eip155` network entries binding the ed25519 operator
identity to the submitter and admin accounts, with client
verification comparing the manifest's declared admin against the
contract's exposed one, as the
[notary BNB profile](notary-bnb.md#the-operator-manifest-already-exists)
requires. A deployment absent from the manifest does not exist, no
matter what is on the chain.

## The plan, in phases

Strictly after the [Stellar phases](charity-stellar.md#the-plan-in-phases)
prove the obligations against real campaigns:

1. **Profile** — `UI-Charity-BNB.md` in `onym-system`, leaning on the
   merged notary BNB profile for every shared EVM rule.
2. **Circuits** — the eligibility predicate families as BN254
   constraint systems, cross-checked against the BLS12-381 circuits
   with shared logical vectors.
3. **Contracts** — the Solidity charity notary contracts plus
   generated verifiers, deployed immutably, identified by chain ID +
   address + runtime code hash.
4. **Relayer EVM backend** — reusing whatever the notary BNB build
   has produced by then; charity must not fork its own EVM plumbing.
5. **Manifest + discovery + conformance** — declare, list, and prove,
   in that order, before any real campaign binds this deployment.

## Honest status

- **Nothing on this page runs.** No profile document, no BN254
  charity circuits, no Solidity, no EVM charity endpoints, no
  fixtures.
- **Both dependencies are themselves plans.** The
  [Stellar charity build](charity-stellar.md) has no code, and the
  [notary EVM backend](notary-bnb.md) this plan reuses is unbuilt —
  this is a plan two plans deep, and it is listed so the dependency
  order is on record, not because work is imminent.
- The abstract contracts it answers to (`Charity.md`,
  `UI-Charity.md`) are merged drafts (0.1, August 2026).

## Next steps

- [Charity](charity.md) — the abstract seat this binds.
- [Charity — Stellar](charity-stellar.md) — the reference plan this
  one re-proves.
- [Notary — BNB Chain](notary-bnb.md) — the merged EVM profile whose
  rules and relayer backend this plan inherits.
