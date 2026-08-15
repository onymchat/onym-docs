# Charity

Charitable aid runs on trust, and trust is exactly what the people
involved can't afford to spend. A donor wants to know their money
reached a real, verified need — but "trust us" is all most programs can
offer. A recipient needs help *now* — but asking for it usually means
handing their identity, their documents, and their situation to the
internet forever. The charity seat exists so that neither has to.

Here's the whole idea in one paragraph: money can keep moving through
ordinary, regulated rails — bank accounts, invoices, real compliance —
while the blockchain holds something different: a **pseudonymous proof
trail** that the procedure was followed. Every case gets a pseudonym,
every allocation gets a proof, every payout gets a hashed reference,
and the chain records *that the rules were obeyed* — a verified case, a
second-person approval, an allocation that fits the pool — without
recording *who anyone is*. The donor can verify the trail
independently. The recipient never appears on it. **Zero PII goes
on-chain, ever.**

**Contract:** [`charity/Charity.md`](https://github.com/onymchat/onym-system/blob/main/charity/Charity.md)
— the technology-free boundary this page summarizes — and
[`charity/UI-Charity.md`](https://github.com/onymchat/onym-system/blob/main/charity/UI-Charity.md),
the messenger-side application profile. Both are drafts.
**Implementations:** [Stellar/Soroban](charity-stellar.md) ·
[BNB Chain](charity-bnb.md) — both are **plans**; no charity code
exists in any Onym repository yet.

This page stays deliberately free of any one blockchain, and of any
one aid program. Like the [notary](notary.md), charity is a role: the
seat splits into two abstract contracts — **Ask for Aid** and
**Donate** — and any chain that can verify the proofs can hold them.

## The words you'll keep seeing

| Term | What it means |
|---|---|
| **Aid pool** | A named fund with a purpose, eligibility rules, a currency, and a balance. Donors fund pools; allocations draw from them. |
| **Case pseudonym** | The opaque identifier a request for aid carries on-chain. It links a case's proof trail together without naming anyone. |
| **Allocation** | A decision to direct a specific amount from a specific pool to a specific case — proposed by one role, approved by another. |
| **Verification attestation** | The operator's signed statement that a case was checked: identity, community connection, reality of need, no duplication. The statement's *hash* is public; its content is not. |
| **Decision hash** | The hash of an approval record. Anyone can verify an allocation was approved; nobody can read the case file from it. |
| **Payment reference hash** | The hash of the real-world payout reference. It ties an on-chain allocation to an off-chain payment without exposing bank details. |
| **Two-person rule** | No single role can both propose and approve an allocation. The proof trail must show two distinct authorized parties. |
| **Operator** | The accountable verification and platform party. Internally it splits into roles — case reviewer, compliance reviewer, allocation approver, finance signer, platform admin — and the split is load-bearing. |

## Contract one: Ask for Aid

A person or a small community in verifiable need applies for help.
The contract governs what happens to that request — and what may never
happen to it.

The applicant describes their situation and provides evidence to the
**operator**, privately. The operator's case reviewer verifies it —
identity, community connection, reality of the circumstances,
reasonableness of the need, absence of duplication. A separate
compliance role clears the payout. An allocation is proposed against a
pool, and a **different** authorized person approves it. The payout
runs over whatever rail the deployment uses. The recipient confirms
receipt, and later confirms whether the help resolved the need.

Every one of those steps leaves a mark on-chain — as a pseudonym, a
hash, or a status transition. None of them leaves a name.

The case moves through one straight line on the happy path:

```text
SUBMITTED → VERIFIED → COMPLIANCE_CLEAR → APPROVED
  → PAID → RECEIVED → IMPACT_CONFIRMED → CLOSED
```

with side branches (`NEEDS_INFO`, `REJECTED`, `COMPLIANCE_HOLD`,
`PAYMENT_FAILED`, `APPEAL`) that never skip a required step. A payout
with no verification attestation, or an approval by the same role that
proposed it, is not a delayed problem — it is an invalid transition
the contract refuses.

What the applicant is promised:

- **The operator sees your case; the world sees a pseudonym.** Your
  documents go to the verifying operator under a stated retention
  policy — never on-chain, never to donors, never in a public record.
- **Financial details come last.** Bank details are collected after
  preliminary approval, so a rejected application never left them
  anywhere.
- **Rejection is private.** A refused or held case shows a typed
  status, not a reason, publicly. The reason goes to you.
- **Your confirmation matters.** A case doesn't close because money
  was sent; it closes when you confirm it arrived, and the trail
  records that distinction.

## Contract two: Donate

A donor — a person, a patron, a fund — gives money to a **pool**, not
to a person. This is the seat's firewall, and it protects both sides:
the donor cannot buy a specific human being, and the recipient never
becomes a line item with a face.

The donor sees a pool's purpose, eligibility, geography, and its
transparency figures — total received, total allocated, number of
cases, available balance. They register a **donation intent**, which
produces a payment reference; the money moves over the deployment's
rail; the operator reconciles the arrival against the intent; and the
funds become available in the pool. The donation's life is also one
line:

```text
INTENT → AWAITING_PAYMENT → PAYMENT_RECEIVED → RECONCILED
  → COMPLIANCE_ACCEPTED → AVAILABLE_IN_POOL
  → PARTIALLY_ALLOCATED → FULLY_ALLOCATED → CLOSED
```

What the donor is promised:

- **You fund criteria, not people.** Allocation to individual cases is
  done by the operator under published verification and allocation
  rules — never by donor selection.
- **You see where it went — in categories, never in names.** An
  allocation shows amount, category, community, country, verification
  status, payment status, and a proof reference. It never shows a
  recipient's identity.
- **Every figure is backed by a proof you can check.** Behind every
  "allocated" number sits an on-chain record — pool ID, allocation ID,
  case pseudonym, decision hash, payment reference hash, timestamp —
  that anyone can verify without the operator's help. The "Verify
  independently" button is optional to press and always there.
- **Reporting is aggregate and honest.** Impact reports state completed
  cases, amounts, geographies, decision times, and confirmation rates.
  Anonymised stories appear only with the recipient's consent.

## Where the zero-knowledge proofs come in

Hashes make the trail *tamper-evident*; zero-knowledge proofs make it
*trustworthy without disclosure*. The implementation profiles bind ZK
proofs to the transitions where "just trust the operator" would
otherwise be the answer:

- **Verified means verified.** The `APPROVED` transition carries a
  proof that a valid verification attestation exists for this case
  pseudonym — without revealing the attestation or the reviewer.
- **Two people really means two.** The proof shows the proposer and
  approver were distinct authorized role keys — without revealing
  which keys.
- **The pool arithmetic holds.** An allocation proves
  `amount ≤ pool available` at the recorded state, so pools cannot be
  silently overdrawn.
- **One payout per allocation.** A nullifier scheme makes a second
  payout proof for the same allocation invalid, on-chain, forever.

The predicates are the contract; the proof system is the profile.
Re-targeting a curve or a chain must not change what is being proven —
the same rule the [notary flavors](notary.md#five-flavors-of-group-state)
already follow.

## The promises, in one place

- **Zero PII on-chain.** No name, document, address, bank detail,
  birth date, or free-text field ever enters calldata, state, or
  events. Negative fixtures must prove it, not policy.
- **Procedure is proven, not asserted.** Verification, second-person
  approval, pool arithmetic, and payout uniqueness are checked by the
  contract, not promised by the operator.
- **The donor firewall is structural.** No operation exists by which a
  donor selects, contacts, or identifies a recipient.
- **The money rail is replaceable.** Bank transfer today, on-chain
  settlement tomorrow — the proof trail's meaning doesn't change with
  the rail, and no rail change touches recorded history.
- **The operator is accountable and inspectable.** Its roles, powers,
  and retention terms are declared in a signed manifest, the same
  pattern the [notary operator](notary.md#the-operator-is-a-clerk-not-a-king)
  and [moderation authority](moderation.md) already use. A power not
  listed is a power it does not have.

## Where the code is

Nowhere, yet — and this book doesn't pretend otherwise.

- **[Stellar/Soroban](charity-stellar.md)** — the planned reference
  implementation: two Soroban contracts riding the existing notary
  relayer, manifest, and proof infrastructure.
- **[BNB Chain](charity-bnb.md)** — the planned EVM profile: Solidity
  contracts with toolchain-generated PLONK **BN254** verifiers,
  sibling to the [notary's BNB plan](notary-bnb.md).

## Next steps

- [Notary](notary.md) — the seat whose relayer, manifest, and proof
  discipline the charity implementations reuse.
- [Discovery](discovery.md) — where charity operators and their
  deployments will be listed once they exist.
