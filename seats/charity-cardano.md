# Charity — Cardano

*Seat implementation page, draft 0.1 — 17 August 2026. Specification:
not written. Code: none — see [Honest status](#honest-status).*

**Profile:** must be written — a `charity/UI-Charity-Cardano.md` in
`onym-system`, binding the
[`charity/Charity.md`](https://github.com/onymchat/onym-system/blob/main/charity/Charity.md)
boundary's **notary and eligibility bindings** to Cardano (Plutus V3,
mainnet and preprod). A third sibling beside the
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

## Where the notary boundary ends

The obligation is unchanged by the chain. What `UI-Charity.md` §8.1
scopes the notary port to — campaign status and revision commitments,
donation and disbursement **receipt commitments**, fund-flow
commitments, **nullifier uniqueness** per campaign and epoch, and
inspectable policy and status changes — is what the validators do,
and nothing else.

In Cardano terms the refusals are sharper than on the EVM, because
the ledger makes value movement so easy to express. Under this
profile ID the validators hold no funds beyond the min-UTXO ADA the
ledger forces every output to carry, mint no assets beyond the
scoped nullifier and beacon tokens described below, pay no addresses,
and execute no transfer of any native asset. An anchored digest
proves *the operator anchored those exact bytes at that time* —
never that money moved or that aid arrived. A stablecoin or ADA
settlement rail would be a separate `financialBindings` profile with
its own finality, refund, and reversal mapping, which this binding
deliberately does not define.

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
on-chain must still prove the nullifier was not spent before. Two
shapes, with a third that only exists as a mitigation:

- **Single registry UTXO per campaign and epoch**, carrying a set or
  trie root in its datum. Correct and simple; each claim spends the
  registry and produces its successor. It also **serialises every
  claim in the scope to one transaction per block**. For a real
  campaign that is not a performance note, it is a liveness failure:
  concurrent claimants collide, and the only way to sustain
  throughput is a single party chaining transactions off-chain —
  which makes the relayer a sequencer and a censorship point the
  design otherwise avoids.
- **Nullifier set as UTXOs** — a sorted on-chain structure (linked
  list or trie) where each node is its own UTXO. Insertion proves
  non-membership by spending the node whose bounds cover the new
  nullifier and producing the covering nodes that replace it.
  Concurrency then scales with the size of the set rather than
  collapsing to one, but two claims falling in the *same gap* still
  contend, so the client needs a rebuild-and-resubmit loop whose
  frequency depends on set density. It also locks min-UTXO ADA in
  every node, permanently, because a nullifier set may never shrink:
  the ADA floor grows linearly with the number of claims ever made
  against the campaign.
- **Shard the registry** by a prefix of the nullifier, dividing
  contention by the shard count at the cost of that many min-UTXO
  floors. Sharding leaks nothing new — the shard index is a function
  of an already-public value — but it is a mitigation for the first
  shape, not a third shape.

Scope encoding forces a second decision. Cardano asset names are at
most 32 bytes, and a BLS12-381 scalar field element already occupies
32 bytes, so a nullifier used as an asset name leaves **no room for a
campaign or epoch prefix in the same name**. Campaign and epoch scope
therefore has to live in the *policy* — a minting policy
parameterized per campaign, whose policy ID is the scope. That is
acceptable only because the campaign identifier is already public;
the profile must state that the policy parameters are derived from
public campaign data alone and never from anything credential-linked,
or the policy ID becomes exactly the cross-campaign identifier
`Charity.md` forbids the nullifier from being.

Which shape the profile picks is an **open question**, and honestly
so: it depends on expected claim concurrency per epoch, which no
deployment has yet measured because no deployment exists.

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

Plutus V3 exposes BLS12-381 as builtins — group operations,
compression, hash-to-curve, and pairing via `millerLoop` /
`mulMlResult` / `finalVerify`
([CIP-381](https://cips.cardano.org/cip/CIP-0381)) — extended with
multi-scalar multiplication
([CIP-133](https://cips.cardano.org/cip/CIP-0133)) and modular
exponentiation ([CIP-109](https://cips.cardano.org/cip/CIP-0109)).
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
against a Rust contract; Cardano needs the verifier written in
Plutus — Aiken or PlutusTx — on top of the builtins above.

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

Whether the verifier fits is separately open. Per-transaction
execution-unit and transaction-size ceilings are governance-set
protocol parameters, and a PLONK verifier's dominant costs are the
multi-scalar multiplication over the commitment set and the final
pairing; CIP-133 exists because naive scalar-multiplication loops in
Plutus exhaust the budget. Whether the `membership-set-v1` verifier
fits one transaction's budget is settled by a prototype and nothing
else. If it does not, the alternatives all cost something the profile
must weigh openly: shrink the circuit, split verification across
chained transactions against a partially-verified state UTXO —
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

A Plutus validator fails. It does not return a four-byte selector,
and there is no Cardano equivalent of the BNB profile's
`StaleCampaignRevision` / `NullifierUsed` / `InvalidProof` surface
observable from the chain. Worse, the failure is not free: a
transaction that fails phase-2 script validation **forfeits the
submitter's collateral**, and the submitter is the operator's relayer
account. Off-chain classification is therefore not a UX nicety on
this chain; it is what keeps a client-side bug from burning operator
ADA.

The client behavior half of the BNB table survives; the place it is
decided does not.

| Class | Condition | Where it is decided | Client behavior |
|---|---|---|---|
| refresh-and-rebuild | Stale campaign revision; epoch rolled over; the covering registry node was consumed by another claim | Off-chain: relayer resolves current state and evaluates the built transaction before submitting | Re-resolve, re-consent to the new revision, rebuild the presentation (a new epoch derives a new nullifier), resubmit. Contention is in this class and is expected, not exceptional. |
| terminal scoped refusal | The nullifier is already present in the set | Off-chain read of the set; on-chain the transaction is simply unbuildable | Show the scoped refusal. Never a person identifier, never a retry. |
| refuse-as-defect | Invalid proof, out-of-field value, unknown policy, missing operator signature | Off-chain evaluation; on-chain it is an untyped script failure | Generator or tooling bug; retrying cannot help. Never submit — submission burns collateral. Proof diagnostics stay private. |
| security event | An anchor contradicting a previously observed anchor; an anchor that disappears in a rollback | On-chain observation after inclusion | Preserve evidence, raise the incident path, never silently resubmit or overwrite. |

[CIP-57](https://cips.cardano.org/cip/CIP-0057) blueprints are the
candidate mechanism for making this mapping mechanical: the validator
publishes its named failure conditions and its datum/redeemer
schemas, and the off-chain evaluator maps a trace to a class from the
declaration rather than from a string match.

The honest cost, which the profile must not bury: on BNB a client can
derive the retry class **from the chain itself**, because the
selector is in the receipt. Here the class is the relayer's claim
about a transaction it chose not to submit. A client can verify the
resulting *state* independently, but not the classification. That is
strictly less verifiable than the EVM sibling, and it is a
consequence of the ledger model, not of a design shortcut.

## Every surface that can carry bytes

The EVM binding's PII fixture greps emitted logs and written storage
slots. Cardano has neither, and has more surfaces instead. There is
no event log at all — **the public trail *is* the UTXO set and its
datums**, which explorers decode and display by default. The
enumeration the profile must extend `Charity.md` §15 item 7 across:

| Surface | Carries | Rule under this profile ID |
|---|---|---|
| Inline datums (CIP-32) | Anchor contents, registry roots, campaign state | Commitments, digests, scoped nullifiers, statuses, timestamps only — typed, with no free-text field |
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
script hash commits to**, all verified before first use. It is
stronger in one respect: parameters are hashed into the script hash,
so a client that recomputes the hash learns what the deployment can
do without trusting any getter. Reference scripts
([CIP-33](https://cips.cardano.org/cip/CIP-0033)) do not change this
— they are a size optimisation, and a client must check that the
referenced script hashes to the declared value rather than trusting
the pointer. Proxy and upgrade patterns are prohibited under this
profile ID; changing a parameter produces a different script hash,
which is a different deployment that must be separately declared.

**Settlement.** Ouroboros Praos settles probabilistically. There is
no `finalized` tag to reconcile against as there is on BSC:
immutability arrives at the security parameter k = 2160 blocks —
roughly twelve hours at mainnet's active slot coefficient — and
everything before that is a depth-based confidence judgment. The
client-visible distinction the siblings require survives, with the
threshold becoming a declared policy rather than a chain guarantee:
`included` (in a block) and `settled` (at or beyond the profile's
declared depth) stay distinct states, and nothing is shown as final
before the second. Naming that cost plainly: BSC's finalized tag is
the chain's statement; a depth threshold is the profile's. Whether
[CIP-140 (Ouroboros Peras)](https://cips.cardano.org/cip/CIP-0140)
should supply a settlement signal once available is an open question.

A rolled-back anchor is a `conflicting_state` **security event, not a
retry**, unchanged from both siblings. Cardano adds one wrinkle worth
stating: a rollback also un-spends the nullifier, so the rebuild path
terminates in either an idempotent identical anchor or the terminal
scoped refusal, both correct — and the event is still reported.

**Receipts.** The BNB fee-bump problem does not arise, because
Cardano has no same-nonce replacement; a transaction lands or expires
at its validity interval. A different instability replaces it: an
expired transaction is rebuilt against different inputs and gets a
different hash for the same operation. The stable reconciliation key
remains the client's operation ID, for a new reason.

**Collateral and min-UTXO.** Plutus transactions require collateral
inputs. The relayer's operator account provides them and **no
end-user Cardano key ever exists**, mirroring the
[Stellar relayer model](notary-stellar.md) exactly. Two operational
consequences the Stellar and EVM backends do not have: concurrent
in-flight transactions need *distinct* collateral UTXOs, so the
relayer must manage a UTXO pool rather than a nonce or sequence
number; and every registry node, beacon UTXO, and anchor output locks
min-UTXO ADA that is never recovered, so the deployment's ADA floor
grows with the number of claims ever made. Who funds that floor, and
whether anchors are stored as UTXOs at all rather than as consumable
datum commitments, is an open profile question with a direct privacy
consequence — a consolidation strategy is also a linkability
strategy.

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
reproducibility, and it presumes a client willing to compile. Whether
clients can be expected to do this, or whether the profile must pin a
published reproducible build artifact and reduce the check to a hash
comparison, is an open question this page cannot settle.

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
  no `UI-Charity-Cardano.md` — verified in this session, `onym-system`
  holds exactly `Charity.md`, `UI-Charity.md`, and
  `UI-Charity-BNB.md`. There are no charity validators, no charity
  circuits on any curve, and no Cardano endpoints in the relayer —
  verified against
  [`onym-contracts`](https://github.com/onymchat/onym-contracts) and
  the relayer repository, neither of which contains the string
  `charity` in any source file.
- **One dependency is already built and reused, not rebuilt:** the
  BLS12-381 TurboPLONK prover backend in `onym-contracts` that the
  Stellar notary runs today. This binding needs new *circuits* on
  that curve, not a new prover.
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

1. **Profile** — write `charity/UI-Charity-Cardano.md`: the
   authorization mechanism, the nullifier-set shape, datum and
   redeemer schemas with their public fields, the off-chain error
   classification and its CIP-57 declaration, deployment identity and
   the recompute check, the settlement-depth threshold, and the
   extended negative-PII fixture set over all six surfaces above.
   `UI-Charity.md` §8.3 sets the bar and it applies unchanged: until
   that document exists and passes conformance tests, "uses Cardano"
   is an implementation choice, not evidence that the boundary is
   satisfied.
2. **Circuits** — the `membership-set-v1` constraint system over
   BLS12-381, setup, and verifying keys. Shared with the Stellar
   charity binding: whichever build reaches them first implements
   them for both.
3. **Prover** — nothing to build. The BLS12-381 backend exists and
   ships in the notary's mobile FFI today; this is the one place
   Cardano starts ahead of the EVM sibling, which still needs a BN254
   backend built.
4. **Verifier prototype, before the validators** — out of order on
   purpose. Whether a BLS12-381 TurboPLONK verifier fits a Plutus
   execution budget is the question that decides whether the rest of
   the chain is worth building, and it can be answered by a
   throwaway script against a published verifying key.
5. **Validators** — the charity validators plus the audited verifier,
   deployed at a parameterized script hash with no upgrade path.
6. **Relayer Cardano backend** — transaction building, UTXO and
   collateral pool management, off-chain evaluation and error
   classification, rollback watching. New; no other binding shares
   it, and if a Cardano notary binding is ever written it inherits
   this rather than the reverse.
7. **Declare, list, prove** — manifest entries, discovery listing,
   and the fixture suite green, in that order, before any real
   campaign binds this deployment.

## Open profile questions

Collected, because each is a place this page refused to invent an
answer:

1. **Nullifier-set shape** — single registry UTXO, UTXO-per-node
   sorted structure, or sharded registry. Decided by expected claim
   concurrency per epoch, which is unmeasured.
2. **Verifier feasibility** — whether the pairing check and MSM fit
   one transaction's execution-unit budget, and which fallback
   applies if not. Settled by prototype (build step 4), not by
   argument.
3. **Min-UTXO funding and anchor storage** — who funds a floor that
   grows with claim count, and whether anchors persist as UTXOs at
   all; consolidation strategy is also linkability strategy.
4. **Double-satisfaction mitigation** — positional output binding
   versus a one-script-input-per-transaction rule, and what the
   latter costs if anchor batching later becomes a required privacy
   mitigation.
5. **Settlement depth** — the declared threshold, and whether
   Ouroboros Peras supplies a chain-level signal once available.
6. **Manifest verification** — recompiled script hash versus a pinned
   reproducible build artifact, and which is realistic for a mobile
   client.
7. **Statement separation from Stellar** — with a shared curve,
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
