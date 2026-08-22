# Charity — Cardano

*Seat implementation page, draft 0.2 — 22 August 2026.*

**Status:** Plan; profile and code not implemented. See
[Honest status](#honest-status).

**Profile:** must be written as `charity/UI-Charity-Cardano.md` in
`onym-system`. It must bind the
[`charity/Charity.md`](https://github.com/onymchat/onym-system/blob/main/charity/Charity.md)
notary and eligibility boundaries to Cardano at protocol version 11 or later.
Mainnet and preprod are in scope; preview is not. This third sibling neither
gates nor is gated by the [Stellar plan](charity-stellar.md) or
[merged BNB specification](charity-bnb.md). **Code:** none.

Cardano lacks the three EVM mechanisms used most by BNB: `msg.sender`, mutable
mappings, and typed revert selectors. This plan names Cardano-native
replacements and their costs. Unresolved choices remain open.

The structural primitives are mature Babbage features: reference inputs,
inline datums, and reference scripts. The verifier primitives are newer.
Pairing arrived with Plutus V3; practical verifier primitives arrived under
protocol version 11, weeks before this page. Claims depending on that second
tier remain conditional.

## At a glance

This is a design and feasibility document.

- **Boundary:** notary and eligibility only; validators would not hold or
  settle charitable funds.
- **Authorization:** required signers parameterized into the script hash are
  the leading `msg.sender` replacement.
- **Nullifiers:** a single registry trie is the proposed pilot; UTXO-per-node
  is the contention fallback.
- **Proofs:** the BLS12-381 prover may be reusable; a Plutus prototype must
  first prove transcript compatibility and budget feasibility.
- **Delivery:** nothing exists; work starts with the verifier prototype and
  ends with an audited preprod pilot.

Architecture review should start with
[nullifier uniqueness](#nullifier-uniqueness-is-the-hard-problem),
[the proof system](#bls12-381-the-third-binding-adds-no-fourth-curve), and
[deployment identity and settlement](#deployment-identity-settlement-and-collateral).
Implementation starts with [Build order](#build-order) and
[Open profile questions](#open-profile-questions).

## Where the notary boundary ends

The chain does not change the boundary. Under `UI-Charity.md` §8.1, validators
handle only campaign status and revision commitments, donation and disbursement
**receipt commitments**, fund-flow commitments, **nullifier uniqueness** per
campaign and epoch, and inspectable policy and status changes.

Under this profile ID, validators hold only ledger-mandated min-UTXO ADA. They
mint only the scoped nullifier and beacon tokens described below, pay no
addresses, and transfer no native asset. An anchored digest proves *the
operator anchored those exact bytes at that time*. It never proves money moved
or aid arrived. Stablecoin or ADA settlement requires a separate
`financialBindings` profile with its own finality, refund, and reversal mapping.
This binding does not define one.

Screening occurs **off-chain, at credential issuance, by the party holding that
authority**. This includes organization checks and any sanctions or identity
screening before issuing an eligibility credential. The chain receives proof
of a predicate, never the screening material. Case material therefore has no
path into a datum, redeemer, or token name, allowing exhaustive negative-PII
fixtures.

## Authorizing an operator write without a sender

Required signers over a parameterized key hash are the working answer.
Cardano transactions expose inputs, outputs, minting, signatures, and required
signers, but no sender. Three candidate gates use those surfaces differently.

| Mechanism | Gate | Cost |
|---|---|---|
| **Required signers** (`extra_signatories`) | Assert that the operator-admin key hash appears in required signers. The key hash is a script parameter committed by the script hash | Rotation changes deployment identity and requires a separately declared deployment. Pinning the hash pins who may write |
| **Beacon / authority token** | Operator writes must spend or reference an NFT | A transferable bearer capability with no on-chain recovery if lost or stolen. One UTXO also serialises every operator write |
| **Script parameterization alone** | Bake campaign scope and admin identity into parameters without a runtime signer check | Insufficient: parameters select the script, not the transaction builder |

Beacon tokens remain available for state-thread identity, not authority.
Required signers most closely match BNB's `immutable` bytecode admin: both pin a
hash that commits to write authority, and both make key rotation a new
deployment.

Proof-authorized writes need no gate. `anchorAidClaim` is sender-agnostic by
default because Cardano has no sender. Validity comes from the eligibility proof
and the validator's campaign state, matching the merged BNB profile.

## Nullifier uniqueness is the hard problem

**Cardano does not enforce one mint per asset name.** A policy can mint the
same policy-ID/asset-name pair again. Uniqueness is a property a minting policy
must *construct*. The standard construction parameterizes the policy by a
specific UTXO that the minting transaction must consume, which is one-shot
precisely because a UTXO can only be spent once. It does not generalise to one
token per arbitrary nullifier value, because the nullifier is unknown when the
policy is parameterized.

So a nullifier token can *represent* a spent nullifier, but something on-chain
must still prove the nullifier was not spent before.

Claims read campaign status, revision, and policy as **reference inputs** under
[CIP-31](https://cips.cardano.org/cip/CIP-0031). They never spend campaign state.
Only the nullifier structure is spent. Otherwise all claims would contend on
one campaign UTXO, making the nullifier comparison meaningless.

| Shape | State and uniqueness | Cost |
|---|---|---|
| **Single registry UTXO per campaign and epoch** | Store a sparse Merkle trie root, using the ecosystem's Merkle Patricia Forestry construction, in the datum. Supply a non-membership witness in the redeemer | O(1) storage and constant locked ADA. Every claim spends and replaces the registry, serialising the scope to one transaction per block. At scale, a relayer chaining transactions becomes a sequencer and censorship point |
| **Nullifier set as UTXOs** | Use a sorted linked list with one UTXO per node. Insert by spending the node covering the new value and producing replacements | Contention spreads with the set, though claims in the same gap still collide and require rebuild-and-resubmit. Every node locks min-UTXO permanently, so ADA grows linearly with all historical claims |
| **Shard the registry** | Partition by a prefix of the already-public nullifier | Divides contention by shard count while multiplying min-UTXO floors. This leaks nothing new and is a mitigation, not a third shape |

Registry contention is a liveness failure, not a safety one: sustaining more
than one spend per block requires someone to chain transactions off-chain, which
is where the relayer acquires that role. The linked list's ADA floor is
permanent because a nullifier set never shrinks, and its rebuild frequency
depends on set density.

Two separate mechanisms are mandatory:

- **Domain separation** requires a per-campaign minting policy. The profile must
  require derivation from public campaign data only. Cardano asset names are
  at most 32 bytes, already filled by a 32-byte BLS12-381 scalar nullifier.
  Campaign and epoch cannot also fit,
  so the policy ID carries scope. Credential-linked parameters would turn the
  policy ID into the forbidden cross-campaign identifier.
- **Uniqueness** uses a campaign- and epoch-scoped registry transition. Its
  datum commits to the spent set, proves non-membership before insertion, and
  changes atomically with claim anchoring. A public minted token marks spending;
  it cannot enforce prior non-membership.

The profile must choose the shape before datum schemas and validators are
written. No deployment exists to measure concurrency, so the answer is
conditional: **the single registry trie is the pilot shape**.

One registry supports one claim per block. At mainnet's roughly twenty-second
block time, that is on the order of a hundred claims an hour. A first campaign
serving tens of beneficiaries over a year would see claims hours or days apart,
orders of magnitude below that ceiling, colliding essentially never. The
relevant threshold is burst size, especially at epoch rollover when every
eligible claimant can act simultaneously.

The profile must define a same-block arrival threshold at rollover. Above the
capacity of one spend per block plus a bounded rebuild loop, the documented
migration is UTXO-per-node. Migration is a new deployment under this profile's
identity rules, not an in-place schema change: a cost worth stating up front
rather than discovering.

## Atomicity comes free, contention does not

Nullifier consumption and claim anchoring occur in one transaction. Every
validator sees the same `ScriptContext`, including transaction outputs. The
nullifier validator can require the claim-anchor output in that transaction.
EVM obtains the same property only by performing both writes in one call.

Contention fails during construction. A Cardano transaction targets specific
UTXOs and is validated against ledger state at inclusion. If another claimant
first consumes the registry node, the transaction spends a nonexistent input
and never becomes valid. This build-time failure is not a revert and drives the
error taxonomy below.

## BLS12-381: the third binding adds no fourth curve

Cardano has exposed BLS12-381 group operations, compression, hash-to-curve, and
pairing through `millerLoop`, `mulMlResult`, and `finalVerify` since the Chang
hard fork ([CIP-381](https://cips.cardano.org/cip/CIP-0381)). Multi-scalar
multiplication ([CIP-133](https://cips.cardano.org/cip/CIP-0133)) and modular
exponentiation ([CIP-109](https://cips.cardano.org/cip/CIP-0109)) arrived with
the **van Rossem hard fork, protocol version 11**: Preview on 8 May 2026 and
mainnet on 18 July 2026, one month before this page.

The profile must carry two consequences. First, it must declare **PV11 or
later**, not merely Plutus V3, beside network magic and script hash,
including in manifest network entries. PV11 exposes
every builtin across Plutus V1, V2, and V3, so language version no longer
identifies capabilities. Second, these builtins and their cost model are months
old, not settled infrastructure. A prototype, not an assumption, must establish
feasibility.

This is **the same curve as Stellar**, whose
[notary already runs in production](notary-stellar.md). The reusable component
is the prover. `plonk/prover` and `sep-*-ffi` in
[`onym-contracts`](https://github.com/onymchat/onym-contracts) generate
BLS12-381 TurboPLONK proofs today. Cardano avoids waiting for the BN254 backend
still needed by the [BNB specification](charity-bnb.md) and
[notary EVM plan](notary-bnb.md).

The verifier is new. Soroban uses host functions and a Rust contract; Cardano
needs a verifier over Plutus builtins. Language choice determines audit surface.
**Aiken** is the working answer because BLS12-381 verifier prior art exists and
its source-to-script-hash compiler path is small enough to reproduce. PlutusTx
adds GHC-plugin complexity to the path a reviewer must trust. The profile must
decide; leaving the language implicit would silently expand the audit surface.

BNB chose BN254 to obtain **toolchain-generated** Solidity verifiers, avoiding
a bespoke audit of handwritten verification. It paid for a second curve,
second setup, and bidirectional cross-curve rejection fixtures. Cardano reuses
the curve but accepts BNB's rejected cost: a handwritten pairing verifier,
without a mature generated-verifier toolchain, deciding claims on real aid.
**That is the load-bearing cost of this binding.** Fixtures do not retire it;
an audit does.

Shared-curve prover reuse also depends on the **Fiat–Shamir transcript**. The
verifier must reproduce it byte-for-byte. Plutus has SHA-2, SHA-3, blake2b,
keccak-256, and RIPEMD-160 builtins, but **no Poseidon**. Implementing a
Poseidon transcript in Plutus field arithmetic would be an order-of-magnitude
problem untouched by MSM optimization.

Sharing a curve is necessary but not sufficient for sharing a prover. Transcript
compatibility is the deciding condition.

The existing transcript is not Poseidon. It is **keccak-256** over a 32-byte
state in `plonk/prover/src/circuit/plonk/transcript.rs`, ported from jf-plonk's
`SolidityTranscript`. Solidity and Soroban therefore agree on bytes, and Plutus
has `keccak_256` under [CIP-101](https://cips.cardano.org/cip/CIP-0101).
Poseidon remains inside the circuit for nullifiers and membership; the verifier
does not recompute it. Prover reuse survives inspection because a decision made
for EVM also fits Plutus. It still requires a byte-level prototype.

Compatibility is plausible; budget remains unknown. Governance-set transaction
limits must fit commitment-set MSM, one `millerLoop`/`finalVerify` pair, the
keccak transcript, and scalar-field linearisation and evaluation aggregation.
Only the last lacks a builtin. Before PV11, naive Plutus MSM above 129 points
could not fit in one transaction. After PV11, no order-of-magnitude blocker is
known, but only measurement can show that the verifier fits.

The profile must weigh three costly failure options: shrink the circuit; split
verification across transactions using a partially verified state UTXO,
**forfeiting single-transaction atomicity**; or change proof systems and lose
the shared prover.

In a three-binding world, cross-curve rejection is not a pair. BN254 proofs must
fail under both BLS12-381 profiles, and BLS12-381 proofs must fail under BNB.
Stellar and Cardano share a curve, so their separation rests entirely on the
statement-tag constant and profile ID. The profile must make
`neg-foreign-statement`-style domain separation load-bearing.

## Errors: the taxonomy moves off-chain

Plutus returns no typed selector. Cardano cannot expose BNB's
`StaleCampaignRevision`/`NullifierUsed`/`InvalidProof` surface from the chain.
A phase-2 script failure also **forfeits the submitter's collateral**, supplied
by the operator relayer. Off-chain evaluation reduces the risk of burning ADA;
it does not remove it. Ledger state can change between evaluation and inclusion,
leaving contention as a normal class. Pre-flight evaluation operates on a
snapshot, so it cannot guarantee that phase-2 failure never reaches the chain.

| Class | Condition | Decision point | Client behavior |
|---|---|---|---|
| refresh-and-rebuild | Stale revision; epoch rollover; another claim consumed the covering node | Relayer resolves current state and evaluates before submission | Re-resolve, re-consent, rebuild the presentation, and resubmit. A new epoch derives a new nullifier. Contention is expected |
| terminal scoped refusal | Nullifier already exists | Off-chain set read; on-chain transaction is unbuildable | Show the scoped refusal; never a person identifier or retry |
| refuse-as-defect | Invalid proof, out-of-field value, unknown policy, missing operator signature | Off-chain evaluation; on-chain untyped failure | Generator or tooling bug. Never submit because submission burns collateral. Keep diagnostics private |
| security event | Contradictory anchor; anchor disappears in rollback | On-chain observation after inclusion | Preserve evidence, raise the incident path, and never silently resubmit or overwrite |

[CIP-57](https://cips.cardano.org/cip/CIP-0057) blueprints are the candidate for
mechanical classification. A validator publishes named failures plus datum and
redeemer schemas; the evaluator maps traces from that declaration, not string
matching.

The profile must disclose that this is strictly less verifiable than BNB. BNB
clients derive retry classes from receipt selectors. Here the class is the
relayer's claim about an unsubmitted transaction. Resulting state is
independently verifiable; classification is not. The client-behavior taxonomy
survives, but its decision point moves
off-chain. The ledger model causes this cost.

## Every surface that can carry bytes

Cardano has no event log. **The public trail *is* the UTXO set and its datums**,
decoded by explorers. Unlike EVM's logs and storage, Cardano exposes more byte
surfaces. The profile must extend `Charity.md` §15 item 7 across:

| Surface | Carries | Rule under this profile ID |
|---|---|---|
| Inline datums ([CIP-32](https://cips.cardano.org/cip/CIP-0032)) | Anchors, registry roots, campaign state | Typed commitments, digests, scoped nullifiers, statuses, and timestamps only; no free text |
| Redeemers | Proof bytes, indices, arguments | Same discipline; never the sealed recipient payload |
| Asset names | 32-byte nullifier | Nothing else; policy ID carries scope |
| Policy IDs / script parameters | Campaign scope, admin key hash | Public campaign data only |
| Transaction metadata (CIP-20, CIP-25) | Arbitrary labeled structures | **Prohibited outright.** No auxiliary data |
| Addresses | Payment and staking parts | Script addresses have no staking part unless declared |

Negative fixtures plant names, IBANs, emails, and addresses in inputs. They grep
every datum, redeemer, controlled-policy asset name, and the entire auxiliary
data field. Passing requires zero hits, no auxiliary data, and no sealed
recipient payload in any datum or redeemer. These fixtures do not exist;
"requires" is the strongest true verb.

Cardano changes four explorer risks from the
[adversary's view](charity-adversary.md):

- Datums are **more** legible than EVM storage because explorers decode them by
  default.
- Nullifier tokens are first-class assets. Per-campaign policy pages index every
  historical claim more conveniently than EVM events.
- The submitter is more exposed. Its pure-ADA collateral UTXO pool creates a
  persistent fingerprint beside its fee-paying address.
- Small-count correlation remains the sharpest edge. eUTXO does not mitigate
  it, and BNB's anchor-batching question remains open here.

## Deployment identity, settlement, and collateral

**Identity.** Clients verify **network magic + script hash + parameter vector +
minimum protocol version** before first use. In scope are mainnet, magic
764824073, and preprod, magic 1, at PV11 or later. Preview, magic 2, is out of
scope, matching opBNB's exclusion from the [EVM sibling](charity-bnb.md).

Parameters are hashed into the script hash, so recomputation reveals deployed
powers without trusting a getter. This is stronger than a getter because fixed
parameters cannot misreport themselves. [CIP-33](https://cips.cardano.org/cip/CIP-0033)
reference scripts do not change identity; clients verify the referenced script
against the declared hash. Proxies and upgrades are prohibited. Parameter
changes create a new script hash and deployment that must be separately declared.

**Settlement.** Ouroboros Praos is probabilistic and has no BSC-style
`finalized` tag. Immutability arrives at k = 2160 blocks, roughly twelve hours
at mainnet's active slot coefficient. Before then, confidence is depth-based.
Clients distinguish `included` from `settled` at the profile's declared depth;
nothing is final before `settled`. BSC finality is a chain statement. Cardano's
depth threshold is a profile statement.

[CIP-140 Ouroboros Peras](https://cips.cardano.org/cip/CIP-0140) targets Q2 2027
in Dijkstra's second phase, behind Linear Leios in Q4 2026. Those dates are
estimates. The declared depth therefore covers the plausible build window.
It is not an interim placeholder. Peras is a possible later optimization, not a
dependency; this binding waits for neither roadmap item.

A rolled-back anchor is a `conflicting_state` **security event, not a retry**.
Rollback also unspends the nullifier. Rebuilding ends in an identical idempotent
anchor or terminal scoped refusal. Both are correct, and the event is reported.

**Receipts.** Cardano has no nonce or same-form fee-bump replacement. A pending
transaction can still be displaced by a transaction spending the same input.
Displacement or validity-interval expiry requires rebuilding with new inputs,
producing a different hash for the same operation. The stable reconciliation
key remains the client operation ID, for a different reason than EVM.

**Collateral and running cost.** Collateral must come from pure-ADA UTXOs held
by the relayer, and **no end-user Cardano key ever exists**, matching the
[Stellar relayer model](notary-stellar.md). Cardano adds three costs:

- Concurrent transactions need distinct collateral UTXOs. The relayer manages
  a UTXO pool, not a nonce or sequence.
- Reference scripts incur a tiered per-byte fee since Conway: 15 lovelace per
  byte on mainnet, multiplied by size tier. A large handwritten verifier is a
  recurring per-transaction cost as well as a larger audit surface. Bigger code
  costs more to audit and more to execute on every claim.
- Pilot locked ADA is constant. UTXO-per-node migration locks min-UTXO per
  nullifier permanently. Anchors increase the floor only if they persist as
  UTXOs instead of consumable datum commitments. That remains open; a
  consolidation strategy is also a linkability strategy.

## The operator manifest

The [live CI-signed manifest](notary-stellar.md#the-live-operator-manifest)
is signed and byte-served today. It would add the Cardano charity profile, its
deployments, and network entries. They bind the Ed25519 operator identity to the
fee-paying Cardano address, collateral policy, and required admin key hash.
**A deployment absent from the manifest does not exist for clients**,
regardless of chain state.

Plutus has no getters, so clients cannot copy BNB's `getOperatorAdmin()` and
`getVerifier()` checks. The Cardano equivalent recomputes the script hash from
the CIP-57 blueprint and declared parameters, then compares it with the on-chain
address. The hash cannot lie about fixed parameters, but this depends on
reproducible builds and a client willing to compile.

A likely mobile alternative publishes configuration in an **inline datum at a
known script address**. An ordinary query recovers getter-like ergonomics.
Anyone compiling once can publicly verify the datum against the script hash.
That avoids compilation by every mobile client without making the datum
self-authenticating. The profile has not chosen among the three options: inline
configuration, per-client recompilation, and a pinned reproducible build
artifact.

## Double satisfaction

Double satisfaction lets one output satisfy two script inputs. Here, spending
two registry nodes for two claims is dangerous. If each validator merely checks
for an expected anchor and continuing node, one output can satisfy both while
two nullifiers are consumed and only one inserted.

The profile must choose between two mitigations:

- Bind each validator to its positional output using an index in the redeemer.
  This is correct and standard, but omission can hide in a passing suite.
- Permit only one script input per transaction. This is easy to audit but
  forbids batching, which may later become an anchor-timing privacy mitigation
  in the [adversary's view](charity-adversary.md).

Fixtures must attempt double satisfaction under the chosen rule.

## Honest status

- **Nothing runs or specifies this binding.** There is no
  `UI-Charity-Cardano.md`. `onym-system` contains exactly `Charity.md`,
  `UI-Charity.md`, and `UI-Charity-BNB.md`. No charity validators or circuits
  exist on any curve, and the relayer has no Cardano endpoints. Neither
  [`onym-contracts`](https://github.com/onymchat/onym-contracts) nor the relayer
  repository contains `charity` in any source file. *(Repository state verified
  17 August 2026.)*
- **One dependency is built and expected to be reused:** the BLS12-381
  TurboPLONK prover used by the Stellar notary. Cardano needs new circuits, not
  a prover, subject to build step 1's transcript check. That result could
  overturn reuse.
- **One dependency is shared and unbuilt:** BLS12-381 charity eligibility
  circuits shared with the [Stellar charity plan](charity-stellar.md), "built
  once, not twice" as in BNB profile §15.
- **Everything else is new and unshared:** Plutus verifier, validators, and the
  Cardano relayer backend.
- `Charity.md` and `UI-Charity.md` are merged draft 0.1 documents from August
  2026. The upstream `Charity.md` §6.8 question about campaign-scoped fields in
  public claim anchors remains unresolved and binds this profile.

## Build order

Cardano has its own dependency chain and gates no sibling.

1. **Verifier prototype — first, not fourth.** Reproduce the prover's keccak-256
   transcript byte-for-byte against an existing proof. Measure whether MSM,
   pairing, and scalar-field arithmetic fit one transaction's PV11 budget. A
   throwaway script and published verifying key answer both. This is the only
   step that can invalidate the profile. Failure forces
   circuit reduction, chained verification, or a new proof system. Each changes
   the proof section, atomicity, and error taxonomy.
2. **Profile, in two halves:** `charity/UI-Charity-Cardano.md`. In parallel with
   step 1, specify authorization; public datum and redeemer schemas; identity
   and recomputation; off-chain errors and CIP-57 declarations; settlement
   depth; and negative-PII fixtures over all six surfaces. Public inputs, proof
   encoding, atomicity, and proof errors wait for step 1. Under `UI-Charity.md`
   §8.3, "uses Cardano" is only an implementation choice until the profile
   exists and passes conformance.
3. **Circuits:** build `membership-set-v1` over BLS12-381 with setup and keys,
   shared with [Stellar](charity-stellar.md). Reuse is asymmetric: Cardano would
   set the public-input interface first because Stellar has no profile. This is
   a coordination dependency, not symmetric reuse. Stellar's author should
   participate rather than inherit an unagreed layout.
4. **Prover:** expected to require no build. The BLS12-381 backend ships in the
   notary mobile FFI, and Plutus can reproduce its keccak-256 transcript.
   "Expected" remains until step 1 confirms byte equality. Cardano starts ahead
   of BNB, which still needs BN254.
5. **Validators:** build charity validators and the verifier, deployed at a
   parameterized script hash without upgrades.
6. **Audit:** independently review the handwritten verifier, validators,
   double-satisfaction binding, and authorization gate. No fixture replaces
   this line item. The profile must name the commissioner, and the result must
   accompany the deployment declaration. Reaching step 8 without it leaves the
   binding's central cost unpaid.
7. **Relayer Cardano backend:** add transaction building, UTXO and collateral
   management, evaluation and error classification, and rollback watching. No
   binding shares it today. A future Cardano notary would inherit this backend,
   not the reverse.
8. **Declare, list, prove:** add manifest entries, discovery listing, then green
   fixtures, in that order, before any real campaign binds the deployment.
9. **Preprod pilot, then mainnet decision:** run real campaigns end to end with
   green conformance. Mainnet follows audit and pilot, matching the
   [Stellar plan](charity-stellar.md). The pilot replaces the estimated
   contention threshold with a measurement.

## Open profile questions

The profile must resolve:

1. **Nullifier-set contention threshold:** the same-block arrival rate at epoch
   rollover, not the average, where the pilot registry stops absorbing
   contention. The registry shape and UTXO-per-node migration are already
   resolved conditionally. The profile states the threshold; the pilot measures
   it.
2. **Verifier feasibility:** whether MSM, pairing, and scalar arithmetic fit one
   PV11 transaction and which fallback applies. Build step 1, not argument,
   decides.
3. **Prover divergence:** keccak-256 and its Plutus builtin support reuse, but
   only byte comparison with a real proof confirms it. Any unreproducible
   verifier-side hash requires a new prover backend and weakens the curve-choice
   argument.
4. **Anchor storage and ADA floor:** whether anchors persist as UTXOs or become
   consumable datum commitments, and who funds them. The pilot registry makes
   the nullifier structure's own cost constant; anchor persistence remains
   open. Consolidation also creates linkability.
5. **Double-satisfaction mitigation:** positional output binding or one script
   input per transaction, including the latter's cost if batching becomes a
   privacy requirement.
6. **Settlement depth:** the declared threshold. Peras targets Q2 2027, so the
   threshold is the build-window answer, not an interim placeholder.
7. **Manifest verification:** inline-datum configuration, client-recompiled
   script hash, or pinned reproducible artifact. The profile must choose what is
   realistic on mobile.
8. **Statement separation from Stellar:** shared-curve separation depends
   entirely on statement tag and profile ID. Fixtures must be designed for this,
   not inherited from a cross-curve pair.

## Next steps

- [Charity](charity.md) — the abstract seat.
- [BNB Chain](charity-bnb.md) — the merged specification whose section shape is
  mirrored but whose choices diverge for the reasons above.
- [Stellar/Soroban](charity-stellar.md) — shares the curve, prover, and unbuilt
  circuits.
- [Solana](charity-solana.md) — the fourth binding, with an external credential
  layer and no reusable prover.
- [Notary — Stellar](notary-stellar.md) — running relayer and manifest
  infrastructure plus the reused BLS12-381 prover.
- [The adversary's view](charity-adversary.md) — the public trail, including
  Cardano-specific datum, token, and collateral differences.
- [Who holds which role](charity-roles.md) — abstract roles mapped to concrete
  parties, including unassigned roles.
