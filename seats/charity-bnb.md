# Charity — BNB Chain (plan)

This page describes an implementation that **does not exist yet** —
and, unlike the [notary's BNB plan](notary-bnb.md), its profile
document is not merged either. Read this as a design intention that
depends on the Stellar charity plan and the notary BNB profile both
landing first.

**Contract:** implements the two abstract contracts of the
[charity seat](charity.md) — **Ask for Aid** and **Donate** — as an
EVM sibling of the [Stellar plan](charity-stellar.md), not its
replacement. A `charity/UI-Charity-BNB.md` profile must be written
and merged in `onym-system` first.
**Code:** none.

## What it will be

The same two contracts — `aid-pool` and `aid-case` — as Solidity on
**BNB Smart Chain** (mainnet 56, testnet 97), with the four charity
predicates re-proved as standard **PLONK over BN254 (KZG)** and
checked by toolchain-generated Solidity verifiers. The reasoning is
inherited wholesale from the [notary BNB profile](notary-bnb.md):
BN254's pairing precompiles have years of production exposure,
verifier generation is automated and audited in widely used
toolchains, gas is cheaper, and no hand-written verifier needs an
audit. The cost is likewise inherited: new constraint systems, a new
setup, new verifying keys, and a second prover backend — a BLS12-381
charity proof is not valid evidence under this profile, and a BN254
proof is not valid under the Stellar one. Fixtures must prove both
rejections.

The EVM hardening rules from the notary profile apply unchanged, and
are restated in the charity profile rather than referenced silently:

- client-chosen 32-byte values (case pseudonyms, intent IDs,
  nullifiers) generated **in-field** by rejection sampling and
  rejected by the contract when out of range;
- verifier and operator-admin addresses as Solidity `immutable`
  values covered by the pinned runtime code hash;
- upgradeable proxies prohibited outright;
- `StaleRevision` and `PublicInputsMismatch` as distinct custom
  errors, because their retry guidance differs;
- transaction hash mandatory in every write receipt from the first
  release, with the operation ID as the stable reconciliation key.

## What the EVM changes for a charity trail

The values on-chain stay exactly as opaque as on Stellar — that
invariant does not move. What moves is the *neighborhood*:

- **Indexing.** BSC calldata, state, and events sit on public,
  heavily-indexed explorers. Pool totals, allocation counts, amounts,
  and timing become trivially chartable by anyone. For a transparency
  trail that is mostly a feature — but the profile must say it, and
  the operator's manifest must declare it as metadata exposure.
- **Amount correlation.** Zero PII on-chain does not mean zero
  inference risk: a public amount plus a public timestamp can
  correlate with off-chain knowledge. The profile must state whether
  amounts appear as plain integers (as on Stellar) or as a proof
  representation, and the choice is per-profile, made explicitly —
  not inherited by default.
- **One submitter.** The relayer's single gas-paying account visibly
  links every case and pool it serves. That is the operator's declared
  role, not a leak — but the selection UI must say so.
- **Finality.** Receipt (`status == 1`) and the `finalized` checkpoint
  are distinct client-visible states; a receipt in a later-orphaned
  block is a security event, not a retry.

## The operator manifest

Same pattern, same file: the live manifest at
`relayer.onym.app/manifest.json` would gain a charity BNB
implementation profile entry, `eip155` network entries binding the
ed25519 operator identity to the submitter and admin accounts, and
the charity role declarations — with client verification comparing
the manifest's declared admin against the contract's exposed one, as
the [notary BNB profile](notary-bnb.md#the-operator-manifest-already-exists)
requires. A deployment absent from the manifest does not exist, no
matter what is on the chain.

## The plan, in phases

Strictly after the [Stellar phases](charity-stellar.md#the-plan-in-phases)
prove the predicates and the state machines against real cases:

1. **Profile** — `UI-Charity-BNB.md` in `onym-system`, leaning on the
   merged notary BNB profile for every shared EVM rule.
2. **Circuits** — the four charity predicates as BN254 constraint
   systems, cross-checked against the BLS12-381 circuits with shared
   logical vectors.
3. **Contracts** — Solidity `aid-pool` and `aid-case` plus generated
   verifiers, deployed immutably, identified by chain ID + address +
   runtime code hash.
4. **Relayer EVM backend** — reusing whatever the notary BNB build
   has produced by then; charity must not fork its own EVM plumbing.
5. **Manifest + discovery + conformance** — declare, list, and prove,
   in that order, before any real pool is funded.

## Next steps

- [Charity](charity.md) — the abstract seat this implements.
- [Charity — Stellar](charity-stellar.md) — the reference plan this
  one re-proves.
- [Notary — BNB Chain](notary-bnb.md) — the merged EVM profile whose
  rules and relayer backend this plan inherits.
