# Charity — Stellar/Soroban (plan)

This page describes an implementation that **does not exist yet**.
There are no charity contracts in [`onym-contracts`](https://github.com/onymchat/onym-contracts),
no charity circuits, and no charity endpoints in the relayer. Read
this as the intended build, in dependency order — not as documentation
of running code.

**Contract:** implements the two abstract contracts of the
[charity seat](charity.md) — **Ask for Aid** and **Donate** — within
the [`charity/Charity.md`](https://github.com/onymchat/onym-system/blob/main/charity/Charity.md)
boundary. A Soroban implementation profile
(`charity/UI-Charity-Stellar.md`) must be written and merged in
`onym-system` before code lands; per
[`UI-Charity.md`](https://github.com/onymchat/onym-system/blob/main/charity/UI-Charity.md),
"uses Soroban" is an implementation choice, not conformance.
**Code:** none.

## What it will be

Two Soroban contracts on Stellar, deployed beside the five notary
contracts and served by the same operator infrastructure:

- **`aid-pool`** (the Donate side) — pool creation with purpose,
  currency, and eligibility metadata; donation intents keyed by an
  opaque intent ID; reconciliation records binding a payment reference
  hash to an intent; running totals (received, allocated, available)
  as public integers.
- **`aid-case`** (the Ask for Aid side) — case pseudonym registration;
  the case state machine as an on-chain enum whose transitions the
  contract enforces in order; allocation records binding pool ID, case
  pseudonym, amount, decision hash, and payment reference hash;
  nullifier storage so one allocation can never anchor two payouts.

Everything the chain stores is a pseudonym, a hash, an amount, a
status, or a timestamp. The contracts have no string fields for case
content, and the profile's negative fixtures must feed PII-shaped
inputs and watch them be rejected.

## Reusing the notary stack

This is the cheapest part of the plan, because the hard infrastructure
[already runs](notary-stellar.md):

- **Relayer.** `relayer.onym.app` gains charity endpoints beside the
  group endpoints. Same model: the operator's account signs and pays;
  no end-user Stellar key ever exists; the relayer's "ok" is transport
  progress, and the client reconciles against the chain before
  believing it.
- **Operator manifest.** The live CI-signed manifest at
  `relayer.onym.app/manifest.json` gains a charity implementation
  profile entry and the charity role declarations — case reviewer,
  compliance reviewer, allocation approver, finance signer — as
  declared powers. A power not listed is a power the operator does
  not have.
- **Proof system.** The proofs target the same TurboPLONK over
  BLS12-381 stack the notary contracts already verify through Soroban
  host functions, with a new set of charity circuits. No new curve,
  no new verifier machinery on this chain.
- **Discovery.** The signed catalog at `discovery.onym.app` lists the
  charity deployment exactly as the manifest declares it, never more.

## The circuits

Four predicates, matching the [seat page's ZK section](charity.md#where-the-zero-knowledge-proofs-come-in):

1. **Attestation exists** — a valid verification attestation, signed
   by an authorized case-reviewer key, exists for this case pseudonym;
   the attestation and the key stay private.
2. **Distinct approver** — the allocation's proposer and approver are
   two different authorized role keys, without revealing which.
3. **Pool arithmetic** — `amount ≤ available` against the pool state
   the proof was built on; a proof built against stale pool state is
   rejected the same way a stale notary epoch is.
4. **Payout uniqueness** — a nullifier derived from the allocation ID,
   spent on first use.

The predicates are specified abstractly so the [BNB profile](charity-bnb.md)
can re-prove them over BN254 without changing their meaning.

## The plan, in phases

Everything below is unbuilt; the order is the dependency chain:

1. **Profile** — write `UI-Charity-Stellar.md` in `onym-system`:
   canonical object mappings, event schemas and their public fields,
   state-machine encodings, error taxonomy, golden vectors against
   `Charity.md`, and the negative-PII fixture set.
2. **Contracts** — `aid-pool` and `aid-case` in `onym-contracts`,
   with the state machines enforced on-chain and hash-anchored
   attestations, decisions, and payment references. This phase is
   useful on its own: a tamper-evident audit trail before any
   circuit exists.
3. **Circuits** — the four predicates as BLS12-381 constraint
   systems; the contracts start requiring proofs on `APPROVED` and
   payout transitions instead of trusting the operator's signature
   alone.
4. **Relayer + manifest** — charity endpoints, role-key management,
   the manifest's charity entries, and the reconciliation reads the
   transparency views depend on.
5. **Transparency surface** — the donor-facing pool figures and the
   "Verify independently" path, reading only what the chain and the
   signed manifest actually say.
6. **Pilot hardening** — real cases end to end on testnet, the
   conformance vectors green, and only then a mainnet deployment
   decision.

Fiat moves over bank rails through the operator for the entire span of
this plan; on-chain settlement (e.g. Stellar USDC payouts) is a later,
separate profile decision and changes nothing about what the proof
trail means.

## Next steps

- [Charity](charity.md) — the abstract seat this implements.
- [Notary — Stellar](notary-stellar.md) — the running infrastructure
  this plan rides on.
- [BNB Chain plan](charity-bnb.md) — the same seat re-proved over
  BN254 for the EVM.
