# Charity — Stellar/Soroban

*Seat implementation page, draft 0.3 — 15 August 2026.*

This page describes an implementation that **does not exist yet**.
There are no charity contracts in [`onym-contracts`](https://github.com/onymchat/onym-contracts),
no charity circuits, and no charity endpoints in the relayer. Read
this as the intended build, in dependency order — not as documentation
of running code.

**Status:** Plan; profile and code not implemented.

**Profile:** must be written — a `charity/UI-Charity-Stellar.md` in
`onym-system`, binding the
[`charity/Charity.md`](https://github.com/onymchat/onym-system/blob/main/charity/Charity.md)
boundary's notary (and later financial) bindings to Soroban.
[`UI-Charity.md`](https://github.com/onymchat/onym-system/blob/main/charity/UI-Charity.md)
§8.3 is blunt about the bar: until that profile exists and passes
conformance tests, "uses Soroban" is an implementation choice, not
evidence that the boundary is satisfied.
**Code:** none.

## What it will be

A Soroban **notary binding** for charity deployments, doing exactly —
and only — what `UI-Charity.md` §8.1 scopes the notary port to:

- campaign status and **revision commitments**, so a client can prove
  which campaign revision was active when an intent was authorized;
- **donation and disbursement receipt commitments** — digests, never
  receipt contents;
- aggregate **fund-flow state** backing a report's
  `sourceCommitment`;
- **nullifier uniqueness** within a campaign and epoch, enforced
  on-chain so a duplicate claim fails as `NULLIFIER_USED` no matter
  which application submits it;
- authorized **policy and destination change** records, so a silent
  operator redirect is detectable; and
- public **audit and report commitments**.

Everything the chain stores is a commitment, a digest, a scoped
nullifier, a status, or a timestamp. The contract's rule that public
state carries only the claim digest, scoped nullifier, and randomized
recipient commitment is enforced by contract types — there are no
string fields for case content — and the profile's negative fixtures
must feed PII-shaped inputs and watch them be rejected. **Zero PII
on-chain** is this profile's concrete rendering of the boundary's
"zero intentional PII publication" invariant.

The **financial binding** is deliberately not on this list. The
charity contracts MUST NOT hold funds, mint assets, or execute
transfers under this binding's profile ID; fiat moves over regulated
rails under the financial provider's own legal authority, and the
chain proves the procedure around it — an anchored receipt digest
proves the operator anchored those bytes at that time, never that
money moved or aid arrived. A Stellar settlement rail (e.g. USDC
payouts) is a later, separate `financialBindings` profile with its
own finality, refund, and reversal mapping — the donation state
machine's `refund-pending`/`refunded`/`reversed` states all need
concrete Stellar semantics before that binding may be declared.
The [merged BNB profile](https://github.com/onymchat/onym-system/blob/main/charity/UI-Charity-BNB.md)
states the same boundary for the EVM side in normative language.

## Reusing the notary stack

This is the cheapest part of the plan, because the hard infrastructure
[already runs](notary-stellar.md):

- **Relayer.** `relayer.onym.app` gains charity notary operations
  beside the group operations. Same model: the operator's account
  signs and pays; no end-user Stellar key ever exists; the relayer's
  "ok" is transport progress, and the client reconciles against the
  chain before believing it.
- **Operator manifest.** The live CI-signed manifest at
  `relayer.onym.app/manifest.json` gains the charity notary profile
  entry. What it declares tracks the `CharityDeployment` object:
  which profile, which contracts, validity, incident contact. A
  binding not declared there does not exist.
- **Proof system.** Eligibility presentations verify against the same
  TurboPLONK over BLS12-381 stack the notary contracts already use
  through Soroban host functions. No new curve, no new verifier
  machinery on this chain.
- **Discovery.** The signed catalog at `discovery.onym.app` lists the
  charity deployment exactly as the manifest declares it, never more.

## The proof obligations

What the chain actually verifies, in the contract's terms:

1. **Eligibility predicates** — an `EligibilityPresentation` proves
   its policy's declared predicate over the declared public inputs,
   bound to the exact campaign revision, without publishing the
   underlying credential. The circuits are per-policy; the profile
   defines which predicate families ship first.
2. **Nullifier scope and uniqueness** — nullifiers are **campaign-
   and epoch-scoped**, stable across revisions within the same
   campaign and epoch (so a policy update cannot enable a second
   claim), and by construction never a cross-campaign or permanent
   beneficiary identifier. Consumption is atomic with claim
   anchoring — the duplicate rule sits at `claim-aid`, exactly where
   the abstract operations table places it, with no separate
   verify-then-consume window. (An earlier revision of this page said
   only "spent on first use", locating the consumption point nowhere;
   the merged BNB profile (summarized on
   [the BNB page](charity-bnb.md)) settles the claim-time reading and
   the Stellar profile must match it.)
3. **Commitment integrity** — receipt and report commitments anchor
   the exact canonical bytes their signed off-chain objects hash to,
   so any application can verify a receipt against the chain without
   trusting the operator's API.

The predicates are the policy's; the proof system is this profile's.
The [merged BNB specification](charity-bnb.md) re-proves the same predicates over
BN254 without changing their meaning.

## The plan, in phases

Everything below is unbuilt; the order is the dependency chain:

1. **Profile** — write `UI-Charity-Stellar.md`: canonical object
   mappings, event schemas and their public fields, state-machine
   encodings, error taxonomy, golden vectors against `Charity.md`,
   and the negative-PII fixture set (`Charity.md` §15 items 1–9).
2. **Contracts** — the charity notary contract(s) in
   `onym-contracts`: revision commitments, receipt commitments,
   nullifier registry, fund-flow anchors. Useful on its own as a
   tamper-evident audit layer before any circuit exists.
3. **Circuits** — the first eligibility predicate family as
   BLS12-381 constraint systems; `present-eligibility` starts
   verifying proofs instead of trusting issuer signatures alone.
4. **Relayer + manifest** — the charity operations, the manifest's
   deployment declarations, and the reconciliation reads
   (`read-donation`, `read-aid-claim`, `read-fund-flow`) that
   clients' watch loops depend on.
5. **Transparency surface** — the fund-flow view and the "Verify
   independently" path, reading only what the chain and the signed
   objects actually prove, with fund flow, allocation, expenditure,
   and impact labeled as the separate claims they are.
6. **Pilot hardening** — real campaigns end to end on testnet, the
   conformance vectors green, and only then a mainnet deployment
   decision. The financial binding question — whether Stellar also
   settles value — is decided after this, not before.

## Next steps

- [Charity](charity.md) — the abstract seat this binds.
- [Notary — Stellar](notary-stellar.md) — the running infrastructure
  this plan rides on.
- [BNB Chain](charity-bnb.md) — the same obligations re-proved over
  BN254 for the EVM, with the merged profile this plan's profile
  must mirror. Neither binding waits for the other to deliver.
- [Cardano](charity-cardano.md) — the third binding, sharing this
  plan's curve, prover, and unbuilt circuits; with a shared curve,
  separation between the two rests on the statement tag alone.
- [Solana](charity-solana.md) — the fourth binding, sharing neither
  this plan's curve nor its proof system, and the only one that
  proposes to settle value on the ledger it notarizes.
- [The adversary's view](charity-adversary.md) — what a public trail
  exposes; most rows apply to Stellar with a smaller explorer
  ecosystem.
