# Charity — Cardano

*Seat implementation page, draft 0.1 — 17 August 2026. Specification:
not written. Code: none — see [Honest status](#honest-status).*

**Profile:** must be written — a `charity/UI-Charity-Cardano.md` in
`onym-system`, binding the
[`charity/Charity.md`](https://github.com/onymchat/onym-system/blob/main/charity/Charity.md)
boundary's **notary and eligibility bindings** to Cardano at protocol
version 11 or later (mainnet and preprod; preview out of scope). A
third sibling beside the
[Stellar plan](charity-stellar.md) and the
[merged BNB specification](charity-bnb.md), gating and gated by
neither.
**Code:** none.

This page does not summarize a profile, because there is no profile.
It makes the case for the binding and enumerates what the profile
will have to decide. Cardano's extended-UTXO ledger has no
`msg.sender`, no mutable mapping, and no typed revert selector — the
three mechanisms the [BNB binding](charity-bnb.md) leans on hardest.
Every section below either states the Cardano-native replacement and
what it costs, or says plainly that the answer is open and why.

The design is specified against the eUTXO ledger rather than
translated from the EVM work, and it draws on two tiers of maturity
that are worth keeping distinct. The structural primitives —
reference inputs, inline datums, reference scripts — are Babbage-era
and long settled. The cryptographic builtins the eligibility verifier
depends on are not: pairing arrived with Plutus V3, and the
primitives that make a verifier practical arrived under protocol
version 11, weeks before this page. Where a claim rests on the second
tier, the page says so.

## Where the notary boundary ends

The obligation is unchanged by the chain. What `UI-Charity.md` §8.1
scopes the notary port to — campaign status and revision commitments,
donation and disbursement **receipt commitments**, fund-flow
commitments, **nullifier uniqueness** per campaign and epoch, and
inspectable policy and status changes — is what the validators do,
and nothing else.

The refusals need stating more sharply than on the EVM, because the
ledger makes value movement so easy to express. Under this profile ID
the validators hold no funds beyond the min-UTXO ADA the ledger
forces every output to carry, mint no assets beyond the scoped
nullifier and beacon tokens described below, pay no addresses, and
execute no transfer of any native asset. An anchored digest
proves *the operator anchored those exact bytes at that time* —
never that money moved or that aid arrived. A stablecoin or ADA
settlement rail would be a separate `financialBindings` profile with
its own finality, refund, and reversal mapping, which this binding
deliberately does not define.

The same boundary runs through compliance, and stating it as a
positive is clearer than stating it as an absence. Screening
obligations — the checks an issuer performs on an organization, and
any sanctions or identity screening performed before an eligibility
credential is issued — are discharged **off-chain, at credential
issuance, by the party holding that authority**. What reaches the
chain is a proof that a predicate holds, never the material the
screening ran on. That is why the negative-PII fixtures below can be
exhaustive rather than best-effort: there is no path by which case
material enters a datum, a redeemer, or a token name in the first
place.

## Authorizing an operator write without a sender

The BNB contract gates operator-attested writes on an
operator-admin address baked immutably into the bytecode and compared
against `msg.sender`. Cardano has no sender: a transaction has
inputs, outputs, a mint field, signatures, and a set of required
signers. Three mechanisms could carry the gate, and they are not
equivalent.

| Mechanism | How the gate works | What it costs |
|---|---|---|
| **Required signers** (`extra_signatories`) | The validator asserts the operator admin's key hash appears in the transaction's required-signers field. The key hash is a **script parameter**, so it is committed to by the script hash. | Rotating the admin key changes the script hash, which changes the deployment identity — a new deployment, separately declared. Strict, and arguably correct: it makes "who may write" part of what a client pins. |
| **Beacon / authority token** | An NFT that operator writes must spend or reference. | Authority becomes a *bearer* capability: whoever holds the UTXO holds the power, transferably, with no on-chain recovery if it is lost or stolen. Spending it also serialises every operator write onto one UTXO. |
| **Script parameterization alone** | Bake the campaign scope and admin identity into the parameters; no runtime check. | Insufficient on its own — parameters constrain *which script* runs, not *who* built the transaction. |

The working answer is **required signers over a parameterized key
hash**, with beacon tokens reserved for state-thread identity (below)
rather than for authority. It is the closest analogue to BNB's
`immutable` admin inside hashed bytecode: in both cases the client
pins a hash, and the hash commits to who may write. What it costs is
key rotation, which on the EVM side is also prohibited — so the cost
is shared, not new.

Proof-authorized writes need no gate at all, which is the one place
eUTXO is simply easier: `anchorAidClaim`'s EVM property of being
sender-agnostic is the Cardano default, because there is no sender to
be agnostic about. Validity comes from the eligibility proof and the
validator's own view of campaign state, exactly as the merged BNB
profile requires.

## Nullifier uniqueness is the hard problem

A widely repeated premise needs correcting before any design rests on
it: **the Cardano ledger does not enforce one mint per asset name.**
Asset names are not unique, and a policy may mint the same
policy-ID/asset-name pair again in a later transaction. Uniqueness is
a property a minting policy must *construct* — the standard
construction being a policy parameterized by a specific UTXO that the
minting transaction must consume, which is one-shot precisely because
a UTXO can only be spent once. That construction does not generalise
to "one token per arbitrary nullifier value", because the nullifier
is not known when the policy is parameterized.

So a nullifier token can *represent* a spent nullifier, but something
on-chain must still prove the nullifier was not spent before.

One precondition first, without which the comparison is meaningless.
The claim validator reads campaign status, revision, and the
registered policy as **reference inputs**
([CIP-31](https://cips.cardano.org/cip/CIP-0031)) — never by spending
the campaign-state UTXO. Under this profile ID campaign state is
referenced and not consumed by a claim, and only the nullifier
structure is spent. Otherwise every claim in a campaign would contend
on one UTXO regardless of nullifier shape, and the comparison below
would be moot. Two shapes, then, with a third that is only a
mitigation:

- **Single registry UTXO per campaign and epoch**, carrying the root
  of a sparse Merkle trie — the Merkle Patricia Forestry
  construction the ecosystem already reaches for — in its datum, with
  non-membership proven by a witness in the redeemer. On-chain
  storage is O(1) and the ADA it locks is *constant*, whatever the
  claim count. Its one problem is contention: each claim spends the
  registry and produces its successor, which **serialises every claim
  in the scope to one transaction per block**. At sufficient scale
  that stops being a performance note and becomes a liveness failure:
  concurrent claimants collide, and the only way to sustain
  throughput past the ceiling is a single party chaining transactions
  off-chain — which makes the relayer a sequencer and a censorship
  point the design otherwise avoids. Where that scale begins is the
  question resolved below.
- **Nullifier set as UTXOs** — a sorted on-chain linked list where
  each node is its own UTXO. Insertion proves non-membership by
  spending the node whose bounds cover the new nullifier and
  producing the nodes that replace it. This trades the registry's
  constant ADA for concurrency: contention scales with the size of
  the set rather than collapsing to one, but two claims falling in
  the *same gap* still contend, so the client needs a
  rebuild-and-resubmit loop whose frequency depends on set density.
  The ADA cost is the mirror image of the trade — every node locks
  min-UTXO permanently, because a nullifier set may never shrink, so
  the floor grows linearly with the number of claims ever made
  against the campaign.
- **Shard the registry** by a prefix of the nullifier, dividing
  contention by the shard count at the cost of that many min-UTXO
  floors. Sharding leaks nothing new — the shard index is a function
  of an already-public value — but it is a mitigation for the first
  shape, not a third shape.

Whichever shape wins, the design rests on separating two jobs the
ledger does not combine on its own — and reading the token as the
enforcement mechanism is the natural mistake this section exists to
prevent:

- **Domain separation** comes from a **per-campaign minting policy**,
  parameterized from public campaign data alone. Scope has to live
  here because it cannot live in the asset name: Cardano asset names
  are at most 32 bytes, a BLS12-381 scalar field element already
  occupies 32, so a nullifier used as an asset name leaves **no room
  for a campaign or epoch prefix beside it**. The policy ID carries
  the scope instead. That is acceptable only because the campaign
  identifier is already public, and the profile must require the
  parameters be derived from public campaign data and never from
  anything credential-linked — otherwise the policy ID becomes
  exactly the cross-campaign identifier `Charity.md` forbids the
  nullifier from being.
- **Uniqueness** comes from an **explicit state transition**, not
  from the token. A campaign- and epoch-scoped registry whose datum
  commits to the consumed set; an update that proves non-membership
  before insertion; and consumption in the same transaction that
  anchors the claim. A minted token marks a spent nullifier
  publicly — it does not, and cannot, enforce that the nullifier was
  unspent, for the reason above.

Which shape the profile picks depends on claim concurrency, which no
deployment has measured because no deployment exists — but that is a
reason to resolve it *conditionally*, not to leave it open until the
validators are already written. The shape fixes the datum schemas and
the validators, so deferring it past those steps is not an option.

The conditional answer: **the single registry trie is the pilot
shape**, and the question only becomes live above a stated threshold.
A registry UTXO sustains one claim per block, which at mainnet's
roughly twenty-second block time is on the order of a hundred claims
an hour — orders of magnitude above what a first campaign serving
tens of beneficiaries over a year generates, where claims arrive
hours or days apart and collide essentially never. The threshold that
matters is not the total beneficiary count but the **burst**, and the
sharpest burst is structural rather than incidental: at an epoch
rollover every eligible claimant becomes able to claim at the same
moment. So the shape question becomes live when expected
same-block arrivals — at rollover, not on average — exceed what one
spend per block plus a bounded rebuild loop absorbs. The profile
states that threshold, ships the registry shape, and treats the
UTXO-per-node structure as the documented migration for a campaign
that exceeds it. A migration is a new deployment under this profile's
identity rules, which is a cost worth stating up front rather than
discovering.

## Atomicity comes free, contention does not

Both siblings require that nullifier consumption and claim anchoring
happen in one transaction, with no window between the eligibility
check and the consumption. eUTXO gives this by construction, and
gives slightly more than the EVM does: a transaction is validated as
a unit, every validator it triggers sees the same `ScriptContext`,
and that context includes the transaction's own outputs. The
validator that consumes the nullifier can therefore check that the
claim anchor output exists in the same transaction — a property an
EVM contract can only get by doing both writes itself.

The cost lands elsewhere. A Cardano transaction is *built* against a
specific set of UTXOs and then validated against the ledger as it is
at inclusion time. If another claimant consumed the covering registry
node first, the transaction does not fail a check — it never becomes
valid, and is rejected as spending a non-existent input. Contention
is a **build-time** failure, not a revert, and that distinction is
what makes the error section below a genuine divergence rather than a
renaming exercise.

## BLS12-381: the third binding adds no fourth curve

Cardano has exposed BLS12-381 as builtins since the Chang hard fork —
group operations, compression, hash-to-curve, and pairing via
`millerLoop` / `mulMlResult` / `finalVerify`
([CIP-381](https://cips.cardano.org/cip/CIP-0381)). The primitives
that make a verifier *practical* are much newer: multi-scalar
multiplication ([CIP-133](https://cips.cardano.org/cip/CIP-0133)) and
modular exponentiation
([CIP-109](https://cips.cardano.org/cip/CIP-0109)) arrived with the
**van Rossem hard fork, protocol version 11** — Preview on 8 May
2026, mainnet on 18 July 2026, one month before this page. That has
two consequences the profile must carry. The deployment precondition
is **network at PV11 or later**, not "Plutus V3", and it belongs
beside network magic and script hash in the identity section and in
the manifest's declared entries; PV11 also made every builtin
available across Plutus V1, V2, and V3, so the ledger language
version no longer identifies a capability set at all. And nothing
below rests on settled infrastructure: these builtins and their cost
model are months old, which is an argument for prototyping, not
against it.

The curve is the one the
[Stellar notary already runs in production](notary-stellar.md), so
this binding proves eligibility over **the same curve as Stellar**
rather than standing up BN254 as the EVM sibling had to.

The strategic point is real but must be stated narrowly. What is
shared is the **prover**: `plonk/prover` and the `sep-*-ffi` crates
in [`onym-contracts`](https://github.com/onymchat/onym-contracts)
already generate TurboPLONK proofs over BLS12-381 for the notary's
group statements, and a Cardano charity binding reuses that backend
instead of waiting on the BN254 backend the
[BNB plan](charity-bnb.md) and the
[notary EVM plan](notary-bnb.md) both still need. What is *not*
shared is the verifier. Soroban verifies through host functions
against a Rust contract; Cardano needs the verifier written on top of
the builtins above.

Which language writes it is not a detail, because it is the largest
single determinant of the audit surface this binding is paying for.
The working answer is **Aiken**: there is existing BLS12-381 verifier
prior art to review against, and the compiler between source and
script hash is small enough that a third party recomputing the hash
(below) is checking something tractable. PlutusTx carries GHC plugin
complexity into exactly that path. The profile decides; leaving it
implicit would be the one place this design silently expands what an
auditor must trust.

That is the trade the page should not soften. BNB chose BN254
specifically to get **toolchain-generated** Solidity verifiers and
avoid a hand-written verifier needing a bespoke audit; it paid for
that with a second curve, a second setup, and cross-curve rejection
fixtures in both directions. Cardano gets the curve for free and pays
in exactly the audit surface BNB refused: a hand-written pairing
verifier, in a language with no mature generated-verifier toolchain,
deciding whether an unnamed person's claim on real aid is honored.
**That is the load-bearing cost of this binding**, and no fixture
retires it — only an audit does.

Sharing a curve is necessary but not sufficient for sharing a prover,
and the condition that actually decides it is the **Fiat–Shamir
transcript**, because that is the one thing a verifier must recompute
byte-for-byte in-script. A transcript over a circuit-native hash
would be fatal here: Plutus offers SHA-2, SHA-3, blake2b, keccak-256,
and RIPEMD-160 as builtins, and **no Poseidon**, so a Poseidon
transcript would have to be written out in Plutus field arithmetic —
an order-of-magnitude problem no MSM optimisation touches.

It is not Poseidon. The notary's transcript
(`plonk/prover/src/circuit/plonk/transcript.rs` in
[`onym-contracts`](https://github.com/onymchat/onym-contracts), a
port of jf-plonk's `SolidityTranscript`) is **keccak-256** over a
32-byte state, chosen so the Solidity and Soroban verifiers agree on
bytes. Plutus has `keccak_256` as a builtin
([CIP-101](https://cips.cardano.org/cip/CIP-0101)). Poseidon appears
in this stack *inside* the circuit — nullifier derivation, membership
— where the verifier never recomputes it. The shared-prover claim
therefore survives inspection rather than resting on an assumption,
and the reason it survives is a decision made for the EVM's benefit
that happens to pay again here.

What remains for the prototype is budget, not compatibility.
Per-transaction execution-unit and size ceilings are governance-set
protocol parameters; the verifier's costs are the multi-scalar
multiplication over the commitment set, one `millerLoop` /
`finalVerify` pair, the keccak transcript, and scalar-field
arithmetic for the linearisation and evaluation aggregation. Only the
last is unassisted by a builtin. Before PV11 this was hopeless — a
naive Plutus MSM of more than 129 points could not fit in a
transaction at all, which is why CIP-133 exists — and after it, no
order-of-magnitude blocker survives inspection. That is not the same
as fitting, and a cost model this new is settled by measurement and
nothing else. If it does not fit, the alternatives all cost something
the profile must weigh openly: shrink the circuit, split verification
across chained transactions against a partially-verified state UTXO —
**which forfeits the single-transaction atomicity established above**
— or change proof systems and lose the shared prover that motivated
the curve choice in the first place.

One consequence for a three-binding world: the cross-curve rejection
fixtures stop being a pair. A BN254 proof must be invalid evidence
under both BLS12-381 profiles, and a BLS12-381 proof must be invalid
under BNB — but Stellar and Cardano share a curve, so separation
between *them* cannot rest on curve mismatch. It rests entirely on
the statement-tag constant and the profile ID, which makes
`neg-foreign-statement`-style domain separation load-bearing here in
a way it is not between BNB and Stellar. The profile must say so.

## Errors: the taxonomy moves off-chain

A Plutus validator fails without returning a selector, so there is no
Cardano equivalent of the BNB profile's `StaleCampaignRevision` /
`NullifierUsed` / `InvalidProof` surface observable from the chain.
The failure is also not free: a transaction failing phase-2 script
validation **forfeits the submitter's collateral**, and the submitter
is the operator's relayer account. Off-chain classification is
therefore not a UX nicety here; it is what *reduces the risk* of a
client-side bug reaching phase-2 validation and burning operator ADA.
It reduces rather than removes, and the profile should say so in
those terms: pre-flight evaluation runs against a snapshot of ledger
state, and state can move between evaluation and inclusion. What
survives that gap is contention, which is why contention is a normal
class below rather than an exceptional one.

The client behavior half of the BNB table survives; the place it is
decided does not.

| Class | Condition | Where it is decided | Client behavior |
|---|---|---|---|
| refresh-and-rebuild | Stale campaign revision; epoch rolled over; the covering registry node was consumed by another claim | Off-chain: relayer resolves current state and evaluates the built transaction before submitting | Re-resolve, re-consent to the new revision, rebuild the presentation (a new epoch derives a new nullifier), resubmit. Contention is in this class and is expected, not exceptional. |
| terminal scoped refusal | The nullifier is already present in the set | Off-chain read of the set; on-chain the transaction is simply unbuildable | Show the scoped refusal. Never a person identifier, never a retry. |
| refuse-as-defect | Invalid proof, out-of-field value, unknown policy, missing operator signature | Off-chain evaluation; on-chain it is an untyped script failure | Generator or tooling bug; retrying cannot help. Never submit — submission burns collateral. Proof diagnostics stay private. |
| security event | An anchor contradicting a previously observed anchor; an anchor that disappears in a rollback | On-chain observation after inclusion | Preserve evidence, raise the incident path, never silently resubmit or overwrite. |

[CIP-57](https://cips.cardano.org/cip/CIP-0057) blueprints are the
candidate mechanism for making the mapping mechanical: the validator
publishes its named failure conditions and its datum and redeemer
schemas, and the evaluator maps a trace to a class from the
declaration rather than from a string match.

The honest cost, which the profile must not bury: on BNB a client
derives the retry class **from the chain itself**, because the
selector is in the receipt. Here the class is the relayer's claim
about a transaction it chose not to submit — the resulting *state* is
independently verifiable, the classification is not. That is strictly
less verifiable than the EVM sibling, and it follows from the ledger
model rather than from a design shortcut.

## Every surface that can carry bytes

The EVM binding's PII fixture greps emitted logs and written storage
slots. Cardano has neither, and has more surfaces instead. There is
no event log at all — **the public trail *is* the UTXO set and its
datums**, which explorers decode and display by default. The
enumeration the profile must extend `Charity.md` §15 item 7 across:

| Surface | Carries | Rule under this profile ID |
|---|---|---|
| Inline datums ([CIP-32](https://cips.cardano.org/cip/CIP-0032)) | Anchor contents, registry roots, campaign state | Commitments, digests, scoped nullifiers, statuses, timestamps only — typed, with no free-text field |
| Redeemers | Proof bytes, indices, operation arguments | Same discipline; the sealed recipient payload never appears |
| Asset names | The nullifier, 32 bytes | Nothing else; scope lives in the policy ID |
| Policy IDs / script parameters | Campaign scope, admin key hash | Derived from public campaign data only |
| Transaction metadata (CIP-20, CIP-25) | Arbitrary labeled structures | **Prohibited outright.** A conforming transaction has no auxiliary data |
| Addresses | Payment and staking parts | Script addresses carry no staking part unless declared |

The negative fixture set therefore plants names, IBANs, emails, and
addresses in the input objects and greps **every datum, every
redeemer, every asset name under every policy the deployment
controls, and the entire auxiliary-data field** — zero hits to pass —
with a separate assertion that auxiliary data is absent entirely
rather than merely clean, and that the sealed recipient payload
appears in no datum or redeemer at all. None of these fixtures exist;
"requires" is the strongest true verb on this page.

What a Cardano explorer adversary sees differs from the BSC row of
the [adversary's view](charity-adversary.md) in four ways, and not
uniformly in Cardano's favour:

- Datums are **more** legible than EVM storage. Explorers decode and
  render inline datums as a matter of course; reading an EVM
  contract's storage takes deliberate effort.
- The nullifier tokens are first-class assets. A per-campaign policy
  ID gives explorers and token trackers a ready-made page listing
  every claim ever made against that campaign — a more convenient
  index than the EVM event trail, offered to the adversary for free.
- The submitter is more exposed. Collateral must be posted from
  pure-ADA UTXOs the operator maintains, so the operator's collateral
  pool is a persistent, linkable on-chain fingerprint alongside the
  fee-paying address.
- Small-count correlation is unchanged, and remains the sharpest
  edge. Nothing about eUTXO mitigates it, and the anchor-batching
  question the BNB profile leaves open is open here too.

## Deployment identity, settlement, and collateral

**Identity.** The Cardano analogue of chain ID + address + runtime
code hash is **network magic + script hash + the parameter vector the
script hash commits to**, plus the **minimum protocol version** the
validators require, all verified before first use. Concretely:
mainnet (magic 764824073) and preprod (magic 1), at protocol version
11 or later; preview (magic 2) is explicitly out of scope, as opBNB
is for the [EVM sibling](charity-bnb.md). Identity here is stronger
than a getter in one respect — parameters are hashed into the script
hash, so a client that recomputes the hash learns what the deployment
can do without trusting anything the deployment says about itself.
Reference scripts ([CIP-33](https://cips.cardano.org/cip/CIP-0033))
do not change identity; a client checks that the referenced script
hashes to the declared value rather than trusting the pointer. Proxy
and upgrade patterns are prohibited under this profile ID; changing a
parameter produces a different script hash, which is a different
deployment that must be separately declared.

**Settlement.** Ouroboros Praos settles probabilistically, and there
is no `finalized` tag to reconcile against as there is on BSC:
immutability arrives at the security parameter k = 2160 blocks —
roughly twelve hours at mainnet's active slot coefficient — and
everything before is a depth-based confidence judgment. The
client-visible distinction survives with the threshold becoming a
declared policy rather than a chain guarantee: `included` and
`settled` (at or beyond the profile's declared depth) stay distinct,
and nothing is shown as final before the second. The cost, plainly:
BSC's finalized tag is the chain's statement; a depth threshold is
the profile's. That is not a temporary state of affairs —
[CIP-140 (Ouroboros Peras)](https://cips.cardano.org/cip/CIP-0140)
sits in the second phase of the Dijkstra roadmap, targeting Q2 2027
behind Linear Leios in Q4 2026, and those dates are published as
estimates. A declared depth threshold is therefore the answer for
this binding's entire plausible build window, not a placeholder, and
Peras is a possible later optimisation rather than a dependency —
nothing in this binding waits for it.

A rolled-back anchor is a `conflicting_state` **security event, not a
retry**, unchanged from both siblings. Cardano adds one wrinkle: a
rollback also un-spends the nullifier, so the rebuild path terminates
in either an idempotent identical anchor or the terminal scoped
refusal, both correct — and the event is still reported.

**Receipts.** The BNB fee-bump problem does not arise in the same
form: there is no nonce and no fee-bump replacement, though a pending
transaction can still be displaced by another spending the same
input, and one that is displaced or expires at its validity interval
is rebuilt against different inputs and gets a different hash for the
same operation. The stable reconciliation key remains the client's
operation ID, for a different reason than on the EVM.

**Collateral and running cost.** Plutus transactions require
collateral inputs. The relayer's operator account provides them and
**no end-user Cardano key ever exists**, mirroring the
[Stellar relayer model](notary-stellar.md) exactly. Three operational
consequences the Stellar and EVM backends do not have:

- Concurrent in-flight transactions need *distinct* collateral
  UTXOs, so the relayer manages a UTXO pool rather than a nonce or
  sequence number.
- Reference scripts are not free per use. Since Conway a tiered
  per-byte fee applies (15 lovelace per byte on mainnet, with a
  multiplier per size tier), so a large hand-written pairing verifier
  is a **recurring per-transaction cost**, not a one-time deployment
  cost. This compounds with the audit argument above: a bigger
  verifier is both more to audit and more expensive to use, every
  time anyone claims.
- Locked ADA is constant under the pilot shape and grows only under
  the migration. The registry trie locks a fixed amount whatever the
  claim count; the UTXO-per-node structure locks min-UTXO per
  nullifier permanently, which is one more reason it is the fallback
  rather than the default. Anchors add to the floor only if they
  persist as UTXOs rather than as consumable datum commitments —
  still open, with a privacy consequence attached, since a
  consolidation strategy is also a linkability strategy.

## The operator manifest

The requirement is the siblings' and does not weaken here: the
relayer's signed, byte-served operator manifest —
[live and CI-signed today](notary-stellar.md#the-live-operator-manifest)
on the Stellar notary side — gains a Cardano charity profile entry,
the deployments it administers, and network entries binding the
ed25519 operator identity to the Cardano payment address that pays
fees, the collateral policy, and the admin key hash the validators
require as a signer. **A deployment absent from the manifest does not
exist for clients**, whatever is on the chain.

The BNB client check — compare the manifest's `adminAccount` against
the contract's `getOperatorAdmin()`, and the declared verifier
against `getVerifier()` — has no direct analogue, because a Plutus
validator exposes no getters. The Cardano equivalent inverts it: the
client **recomputes** the script hash from the published CIP-57
blueprint and the manifest's declared parameters, and checks it
against the hash the on-chain address commits to. Better in one
direction, since no runtime getter can lie about a value the hash
already fixes. Worse in another: the check is only as good as build
reproducibility, and it presumes a client willing to compile.

A third option sits between them and is probably the realistic one
for a mobile client: the deployment publishes its configuration in an
**inline datum at a known script address**, readable with an ordinary
chain query and no compiler. That recovers the ergonomics of a getter
without the ability to lie about the parameters, since the datum can
be checked against the script hash by anyone who *does* compile —
once, publicly, rather than per client. Which of the three the
profile requires is open.

## Double satisfaction

Cardano's characteristic validator vulnerability deserves naming
because this design is exposed to it. Double satisfaction occurs when
one output satisfies the checks of two script inputs in the same
transaction. Here the exposure is concrete: if a transaction spends
two registry nodes for two claims, and each validator checks only
that *an* output carries the expected anchor and *a* continuing node
exists, a single output can satisfy both — consuming two nullifiers
while inserting one.

Two mitigations, with their costs. Bind each validator to its own
output by index, carried in the redeemer and checked positionally —
correct, and the standard construction, but it is the kind of check
whose absence is invisible in a passing test suite. Or forbid more
than one script input per transaction under this profile ID —
trivially auditable, at the cost of batching, which the anchor-timing
mitigation in the [adversary's view](charity-adversary.md) may
eventually want. The profile chooses; the fixture set must include a
deliberate double-satisfaction attempt either way.

## Honest status

- **Nothing on this page runs, and nothing specifies it.** There is
  no `UI-Charity-Cardano.md`: `onym-system` holds exactly
  `Charity.md`, `UI-Charity.md`, and `UI-Charity-BNB.md`. There are
  no charity validators, no charity circuits on any curve, and no
  Cardano endpoints in the relayer — neither
  [`onym-contracts`](https://github.com/onymchat/onym-contracts) nor
  the relayer repository contains the string `charity` in any source
  file. *(Repository state verified 17 August 2026.)*
- **One dependency is already built and expected to be reused, not
  rebuilt:** the BLS12-381 TurboPLONK prover backend in
  `onym-contracts` that the Stellar notary runs today. This binding
  needs new *circuits* on that curve, not a new prover — subject to
  the transcript check in build step 1, which is the one result that
  could overturn it.
- **One dependency is shared and unbuilt:** the BLS12-381 charity
  eligibility circuits, shared with the
  [Stellar charity plan](charity-stellar.md) in the "built once, not
  twice" sense the BNB profile's §15 uses.
- **Everything else is new and shared with nothing:** the Plutus
  verifier, the validators, and the relayer's Cardano backend, which
  no other Onym binding needs today.
- The abstract contracts this would answer to (`Charity.md`,
  `UI-Charity.md`) are merged drafts (0.1, August 2026), and the
  `Charity.md` §6.8 wording question the BNB profile flags upstream —
  campaign-scoped fields in public claim anchors — is unresolved and
  binds this profile too.

## Build order

Cardano proceeds on its own dependency chain. Neither sibling gates
it and it gates neither, which is the house position and not a new
claim:

1. **Verifier prototype — first, not fourth.** It comes before the
   profile because it is the only step that can invalidate the
   profile. Its questions in order: does a Plutus transcript
   reproduce the prover's keccak-256 challenges byte-for-byte against
   an existing proof; and do the MSM, pairing, and scalar-field
   arithmetic fit one transaction's PV11 budget. A throwaway script
   against a published verifying key answers both. Every fallback
   this page lists for a negative answer — shrink the circuit, chain
   verification across transactions, change proof systems —
   invalidates the profile's proof section, its atomicity property,
   and part of its error taxonomy. Writing those first and revising
   them after would be a rewrite disguised as a revision.
2. **Profile, in two halves** — `charity/UI-Charity-Cardano.md`. The
   verifier-independent half can be written immediately and in
   parallel with step 1: authorization, datum and redeemer schemas
   with their public fields, deployment identity and the recompute
   check, the off-chain error classification and its CIP-57
   declaration, the settlement-depth threshold, and the extended
   negative-PII fixture set over all six surfaces above. The
   proof-system half — public-input layout, proof encoding, the
   atomicity guarantee, and the proof-related error classes — waits
   on step 1's result. `UI-Charity.md` §8.3 sets the bar for both and
   applies unchanged: until the document exists and passes
   conformance tests, "uses Cardano" is an implementation choice, not
   evidence that the boundary is satisfied.
3. **Circuits** — the `membership-set-v1` constraint system over
   BLS12-381, setup, and verifying keys, shared with the
   [Stellar charity binding](charity-stellar.md). "Shared" is
   accurate but asymmetric in practice and should not be stated
   otherwise: circuit design depends on a profile's public-input
   layout, Stellar has no profile, and Cardano's would be written
   first — so this binding would *set* the shared circuit interface
   that the Stellar profile then has to match. That is a coordination
   dependency, not a symmetric reuse, and the Stellar profile's
   author should be in the room when the layout is fixed rather than
   inheriting constraints they never agreed to.
4. **Prover** — expected to be nothing to build. The BLS12-381
   backend exists and ships in the notary's mobile FFI today, and its
   keccak-256 transcript is reproducible from Plutus builtins; this
   is the one place Cardano starts ahead of the EVM sibling, which
   still needs a BN254 backend built. "Expected" until step 1
   confirms it at the byte level.
5. **Validators** — the charity validators plus the verifier,
   deployed at a parameterized script hash with no upgrade path.
6. **Audit** — an independent review of the hand-written verifier and
   the validators, and a line item rather than an assumption, because
   this page names that verifier as the binding's load-bearing cost
   and states that no fixture retires it. Its scope is the verifier,
   the double-satisfaction binding, and the authorization gate; the
   profile must name who commissions it and publish the result
   alongside the deployment declaration. A deployment that reaches
   step 8 without it has not paid the cost this binding was designed
   around.
7. **Relayer Cardano backend** — transaction building, UTXO and
   collateral pool management, off-chain evaluation and error
   classification, rollback watching. New; no other binding shares
   it, and if a Cardano notary binding is ever written it inherits
   this rather than the reverse.
8. **Declare, list, prove** — manifest entries, discovery listing,
   and the fixture suite green, in that order, before any real
   campaign binds this deployment.
9. **Preprod pilot, then a mainnet decision** — real campaigns end to
   end on preprod, the conformance vectors green, and only then a
   mainnet deployment decision, matching the
   [Stellar plan](charity-stellar.md)'s final phase. The ordering is
   the point: the mainnet decision **follows** the audit and the
   pilot rather than preceding them, and the pilot is where the
   contention threshold above stops being an estimate.

## Open profile questions

Collected, because each is a place this page refused to invent an
answer:

1. **Nullifier-set contention threshold** — the shape itself is
   resolved conditionally above (registry trie for the pilot,
   UTXO-per-node as the documented migration). What is open is the
   number: the same-block arrival rate, measured at epoch rollover
   rather than on average, at which the registry stops absorbing
   contention. The profile states it; a pilot measures it.
2. **Verifier feasibility** — whether the MSM, pairing, and
   scalar-field arithmetic fit one transaction's PV11 execution-unit
   budget, and which fallback applies if not. Settled by prototype
   (build step 1), not by argument.
3. **Prover divergence** — the residual risk behind the shared-prover
   claim. The transcript hash is keccak-256 and Plutus has the
   builtin, so the claim survives inspection, but only a byte-level
   check against a real proof confirms it. If any verifier-side hash
   turns out to be unreproducible in Plutus, the prover changes, the
   backend stops being shared, and the strategic argument for the
   curve choice weakens accordingly.
4. **Anchor storage and its ADA floor** — the registry shape makes
   the nullifier structure's own cost constant, so what remains open
   is whether anchors persist as UTXOs at all rather than as
   consumable datum commitments, and who funds the floor if they do.
   Consolidation strategy is also linkability strategy.
5. **Double-satisfaction mitigation** — positional output binding
   versus a one-script-input-per-transaction rule, and what the
   latter costs if anchor batching later becomes a required privacy
   mitigation.
6. **Settlement depth** — the declared threshold. Peras is a Q2 2027
   roadmap item, so this is the answer for the whole build window
   rather than an interim one.
7. **Manifest verification** — published inline-datum configuration,
   client-recompiled script hash, or a pinned reproducible build
   artifact, and which is realistic for a mobile client.
8. **Statement separation from Stellar** — with a shared curve,
   separation rests entirely on the statement tag and profile ID; the
   fixture set must be designed for that, not inherited from the
   cross-curve pair.

## Next steps

- [Charity](charity.md) — the abstract seat this would bind.
- [BNB Chain](charity-bnb.md) — the merged specification whose
  section shape this page mirrors and whose choices it diverges from
  with reasons.
- [Stellar/Soroban](charity-stellar.md) — the sibling that shares
  this binding's curve, prover, and unbuilt circuits.
- [Notary — Stellar](notary-stellar.md) — the running relayer and
  manifest infrastructure, and the BLS12-381 prover this binding
  reuses.
- [The adversary's view](charity-adversary.md) — what a public trail
  exposes; the datum, token, and collateral rows above are the
  Cardano-specific deltas.
- [Who holds which role](charity-roles.md) — the abstract roles
  mapped to concrete parties, including the unassigned ones.
