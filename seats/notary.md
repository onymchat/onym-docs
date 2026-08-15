# Notary

Group chat has a quiet problem: *someone* has to keep the group's shared
facts — who's in, what the current membership commitment is, which change
came first. Whoever keeps those facts usually ends up owning the group.
The notary seat exists so that nobody does.

Here's the whole idea in one paragraph: every group's shared state is
just two small things — an opaque cryptographic **commitment** and a
counter called the **epoch**. The notary is a referee that holds them
and accepts a change only when it comes with a **zero-knowledge proof**
that the change is allowed under the governance rules the group picked
at creation. The notary never sees messages, names, or the member list —
it checks math, not people. The app can't cheat because the rules live
in a public contract, not in app code. And the group's choice of referee
is **pinned**: no app update, price change, or catalog edit can quietly
move a group to a different notary.

**Contract:** [`notary/UI-Notary.md`](https://github.com/onymchat/onym-system/blob/main/notary/UI-Notary.md)
— the technology-free boundary this page describes.
**Implementations:** [Stellar/Soroban](notary-stellar.md) (the
reference implementation, running today) ·
[BNB Chain](notary-bnb.md) (a merged design with an implementation
plan — nothing built yet).

This page stays deliberately free of any one blockchain. A notary is
not "the Stellar thing" — it is a role, and the whole point of
specifying it abstractly is that several concrete notaries, on several
chains, run by several parties, can hold it at once.

## The words you'll keep seeing

| Term | What it means |
|---|---|
| **Commitment** | A 32-byte opaque value that stands for the group's membership and secrets. The notary stores it; only members can open it. |
| **Epoch** | The group's revision counter. Every accepted change increments it by exactly one — no gaps, no rewinds. |
| **Flavor** | The governance rule chosen at creation: who is allowed to advance the group's state. Five exist (see below). |
| **Proof** | A zero-knowledge proof that the proposed new commitment follows from the old one under the flavor's rule — without revealing who acted. |
| **Circuit type** | Which proof system and chain family the proofs target. Today `plonk` over BLS12-381 on Stellar; the planned BNB profile uses `plonk-bn254-kzg`. |
| **Binding** | The group's pinned record of *its* notary: chain, contract, flavor, circuit type, genesis evidence. Sticky by design. |
| **Operator** | The party running the relayer service. It pays fees and may gate group *creation* — it can never advance a group's state. |
| **Relayer** | The operator's HTTP front door. It submits your operation to the chain and pays the network fee so your wallet doesn't have to exist. |

## Five flavors of group state

You pick a flavor when you create a group, and it never changes. Pick by
who should be able to advance the group.

| Flavor | Members | Admins | Who advances state |
|---|---|---|---|
| `anarchy` | up to 2¹¹ | — | any member with a valid membership proof |
| `oneonone` | exactly 2 | — | nobody — immutable after creation |
| `democracy` | up to 2¹¹ | — | a K-of-N quorum of members, proven in one batched proof |
| `oligarchy` | up to 2¹¹ | up to 32 | a K-of-N quorum of admins; the admin roster stays hidden after creation |
| `tyranny` | up to 2¹¹ | 1 | one pinned admin, proven without revealing who they are across groups |

The flavor is a governance *predicate*, not a chain feature: an
implementation profile re-expresses the same five rules in whatever
proof system its chain verifies, and re-targeting the curve must not
change what they prove.

## How a state change flows

1. **Read first.** Your app fetches the group's current commitment and
   epoch from the canonical system and verifies what it read.
2. **Prove locally.** Your device builds the new commitment and a
   zero-knowledge proof that the transition is allowed under the
   group's flavor. Secrets and witnesses never leave the device.
3. **Submit through the relayer.** The app sends the operation to the
   operator's relayer, which checks it against its allowlist, pays the
   network fee, signs the outer transaction, and submits it.
4. **The contract decides.** On-chain, the contract compares the
   proof's public inputs to the *stored* state — a proof built against
   stale state is rejected, and a replayed proof is rejected — then
   verifies the proof and, only then, writes the new commitment and
   epoch, emitting an event.
5. **Reconcile before believing.** The app reads back the accepted
   state with evidence before it treats the change as real. The
   relayer saying "ok" is transport progress, not truth.

## Choosing where your proofs land

A notary operator may run deployments on **several blockchains at
once**, and the contract says *you* choose — at group creation,
alongside flavor — which one receives your group's proofs. The
selection is two-dimensional: **which notary operator**, and **which of
that operator's backends** gets the proofs. A multi-chain operator is
several notary deployments under one accountable seat, not one notary
spanning chains — each backend keeps separate state, and a group lives
on exactly one.

The contract requires the choice be shown with its real consequences —
which public ledger your group's (opaque) activity lands on, its fees,
its finality, its metadata exposure — not just a curve name. Once made,
it is pinned into the group binding like everything else. Groups do not
migrate between chains; until a governed migration protocol exists, a
re-created group is a *new* group. A joiner never chooses: the
invitation carries the pinned binding and the joining client verifies
it or refuses. Prover availability is a capability, not a preference —
a client without the right prover refuses to join rather than
substitute evidence.

[Discovery](discovery.md) is how you find the choices. Today it lists
Soroban relayers — a name, a URL, and supported Stellar networks — and
the live signed catalog at `discovery.onym.app` already indexes the
reference relayer's operator manifest. As more notaries appear, it
grows into a catalog of operators run by **different parties**, each
declaring exactly which chains it supports in its signed manifest.
Some will support only one — an honest, declared condition — and the
app offers only combinations your chosen operator actually declares.
It must never paper over a gap by silently switching you to a
different operator.

## The operator is a clerk, not a king

The relayer's operator holds real but narrow powers, and it declares
them in a signed manifest served byte-for-byte at
`GET /manifest.json` — the pattern the
[moderation authority](moderation.md) already uses for its terms.
Declared powers are exhaustive: a power not listed is a power the
operator does not have. Group bindings and entitlements pin the
SHA-256 of the manifest bytes, so changed terms mean a new manifest,
never an edit in place. The governing invariant:

> The operator's key can pay for, submit, and gate the creation of
> group state. It can never author a group transition.

Concretely, the operator may refuse to submit (you can use another
relayer), require payment, rate-limit, and flip the creation gate. It
cannot advance any group's commitment, forge or substitute proofs, move
a group's binding, or make an invalid operation valid by accepting
money for it. These limits are enforced by the contract, not promised
by policy. The reference relayer serves its signed manifest live —
the [Stellar page](notary-stellar.md) documents the pipeline.

## The promises

- **The binding is sticky.** Nothing short of the group's own governed
  migration — not an app update, a default change, a cheaper chain, or
  a new catalog — moves an existing group to a different notary.
- **Every change is proven.** Prior state, next state, group identity,
  and every rule-relevant value are bound into the proof. Stale proofs
  and replayed proofs are rejected on-chain.
- **The submitter is not an oracle.** A relayer's success response is
  never treated as canonical state; the app verifies against the chain.
- **Payment never buys validity.** Paying a provider answers whether it
  will *serve* you, never whether your transition satisfies the rules.
  A paid invalid operation stays invalid.
- **Secrets stay at the edge.** Keys, seed material, and proof
  witnesses live on the device. The chain sees commitments; the relayer
  sees ciphertext-shaped bytes and metadata it must declare.

## Where the code is

- **[Stellar/Soroban](notary-stellar.md)** — the running reference
  implementation: five Soroban contracts on testnet, the relayer at
  `relayer.onym.app`, and the live CI-signed operator manifest. Every
  group that exists today lands here.
- **[BNB Chain](notary-bnb.md)** — the merged implementation profile
  and its phased build plan. The design is settled; the code does not
  exist.

## Next steps

- [Discovery](discovery.md) — how clients find relayers, contracts, and
  (eventually) competing notary operators.
- [Deployment](../deployment.md) — how the reference deployment brings
  the relayer up.
