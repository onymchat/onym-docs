# Charity — Stellar/Soroban

*Seat implementation page, draft 0.3 — 15 August 2026.*

This implementation **does not exist yet**. [`onym-contracts`](https://github.com/onymchat/onym-contracts)
has no charity contracts or circuits, and the relayer has no charity endpoints.
Dependency-ordered plan, not running code.

**Status:** Plan; profile and code not implemented.

**Profile:** must be written as `charity/UI-Charity-Stellar.md` in
`onym-system`, binding the
[`charity/Charity.md`](https://github.com/onymchat/onym-system/blob/main/charity/Charity.md)
notary and later financial boundaries to Soroban. Under
[`UI-Charity.md`](https://github.com/onymchat/onym-system/blob/main/charity/UI-Charity.md)
§8.3, until the profile exists and passes conformance, “uses Soroban” is an
implementation choice, not evidence that the boundary is satisfied. **Code:** none.

## What it will be

A Soroban **notary binding** limited to `UI-Charity.md`
§8.1:

- campaign status and **revision commitments**, proving the revision active at
  authorization;
- digest-only **donation and disbursement receipt commitments**;
- aggregate **fund-flow state** backing report `sourceCommitment`;
- campaign- and epoch-scoped **nullifier uniqueness**, returning
  `NULLIFIER_USED` for duplicates from any application;
- authorized **policy and destination changes**, exposing silent redirects; and
- public **audit and report commitments**.

Chain state contains only commitments, digests, scoped nullifiers, statuses, and
timestamps. Types restrict public state to claim digests, scoped nullifiers, and
randomized recipient commitments; no case-content strings exist. Negative
fixtures must reject PII-shaped inputs. **Zero PII on-chain** implements the
“zero intentional PII publication” invariant.

The **financial binding** is separate. Under this profile ID, charity contracts
MUST NOT hold funds, mint assets, or transfer value. Fiat moves over regulated
rails under the financial provider's legal authority. An anchored receipt
digest proves only that the operator anchored those bytes at that time, never
that money moved or aid arrived.

A later Stellar settlement rail, e.g. USDC, needs a separate
`financialBindings` profile with its own finality, refund, and reversal mapping.
Its `refund-pending`/`refunded`/`reversed` states need concrete Stellar semantics
before declaration. The [merged BNB profile](https://github.com/onymchat/onym-system/blob/main/charity/UI-Charity-BNB.md)
states the same EVM boundary normatively.

## Reusing the notary stack

- **Relayer:** `relayer.onym.app` adds charity operations. The operator signs
  and pays; no end-user Stellar key exists. Relayer “ok” means transport
  progress; clients reconcile against the chain.
- **Operator manifest:** the live CI-signed
  `relayer.onym.app/manifest.json` adds the charity profile. It declares the
  `CharityDeployment` profile, contracts, validity, and incident contact. An
  undeclared binding does not exist.
- **Proof system:** eligibility uses the existing TurboPLONK/BLS12-381 stack
  through Soroban host functions. This adds no curve or verifier machinery.
- **Discovery:** the signed `discovery.onym.app` catalog lists exactly what the
  manifest declares, never more.

## The proof obligations

The chain verifies three properties:

1. **Eligibility predicates.** An `EligibilityPresentation` proves the policy's
   predicate over declared public inputs, bound to the exact campaign revision,
   without publishing the credential. Circuits are per-policy; the profile
   defines which predicate families ship first.
2. **Nullifier scope and uniqueness.** Nullifiers are **campaign- and
   epoch-scoped** and stable across same-scope revisions, so policy updates
   cannot enable a second claim. By construction, they are never a
   cross-campaign or permanent beneficiary identifier. Atomic claim anchoring
   consumes the nullifier at `claim-aid`, with no
   verify-then-consume window. The merged BNB profile resolved an earlier
   “spent on first use” ambiguity; the [BNB page](charity-bnb.md) summarizes the
   claim-time reading that Stellar must match.
3. **Commitment integrity.** Receipt and report commitments bind the canonical
   bytes of signed off-chain objects. Applications can verify them without the
   operator API.

Predicates belong to policy; proof systems belong to profiles. The
[merged BNB specification](charity-bnb.md) re-proves the same obligations over
BN254 without changing predicate meaning.

## The plan, in phases

All phases are unbuilt and dependency-ordered:

1. **Profile:** write `UI-Charity-Stellar.md` with canonical object mappings,
   event schemas and public fields, state-machine encodings, error taxonomy,
   `Charity.md` golden vectors,
   and negative-PII fixtures from §15 items 1–9.
2. **Contracts:** add revision and receipt commitments, the nullifier registry,
   and fund-flow anchors to `onym-contracts`. This tamper-evident audit layer is
   useful before circuits exist.
3. **Circuits:** build the first BLS12-381 eligibility predicate family, making
   `present-eligibility` verify proofs instead of issuer signatures alone.
4. **Relayer + manifest:** add operations, deployment declarations, and
   `read-donation`/`read-aid-claim`/`read-fund-flow` reconciliation reads for
   client watch loops.
5. **Transparency surface:** expose fund flow and “Verify independently” using
   only chain and signed-object evidence. Label fund flow, allocation,
   expenditure, and impact as separate claims.
6. **Pilot hardening:** run real campaigns end to end on testnet with green
   conformance vectors. Decide on mainnet only afterwards; decide whether
   Stellar settles value later still.

## Next steps

- [Charity](charity.md) — the abstract seat.
- [Notary — Stellar](notary-stellar.md) — running reused infrastructure.
- [BNB Chain](charity-bnb.md) — the same predicates over BN254 and the merged
  profile Stellar must mirror; neither binding waits for the other.
- [Cardano](charity-cardano.md) — the third binding; it shares the curve, prover,
  and unbuilt circuits. Statement tags alone separate the two.
- [Solana](charity-solana.md) — the fourth binding; it shares neither curve nor
  proof system and alone proposes same-ledger settlement.
- [The adversary's view](charity-adversary.md) — the public trail; most rows
  apply with a smaller explorer ecosystem.
