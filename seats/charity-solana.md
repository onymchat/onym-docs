# Charity — Solana

*Seat implementation page, draft 0.1 — 21 August 2026.*

**Status:** Plan; profile and code not implemented. See
[Honest status](#honest-status).

**Profile:** must be written — a `charity/UI-Charity-Solana.md` in
`onym-system`, binding the
[`charity/Charity.md`](https://github.com/onymchat/onym-system/blob/main/charity/Charity.md)
boundary's **notary and eligibility bindings** to Solana
(mainnet-beta and devnet; testnet out of scope), and — this is what
makes it different from its three siblings — proposing the first
**financial binding** any charity page has described.
**Code:** none.

There is no profile to summarize, so this page does what the
[Cardano plan](charity-cardano.md) does: it makes the case for the
binding and enumerates what the profile will have to decide. Two
things separate it from the siblings, and both are load-bearing.

The first is that Solana already runs the layer the other bindings
would have to invent. The **Solana Attestation Service** is a
Foundation-maintained public good whose Credential, Schema, and
Attestation accounts carry authorized issuer signers, versioned
schemas, and per-attestation expiry — which is, almost field for
field, what `Charity.md` §6.2 and §6.7 require of an organization
credential and an eligibility policy. Composing with it, rather than
re-deriving it, is the strongest argument for this binding.

The second is that Solana is the first candidate where **settlement
on the same ledger is worth designing now**. The
[Stellar plan](charity-stellar.md) already names a USDC rail as a
later, separate `financialBindings` profile, under the same refund
and reversal precondition — so the option is not new, and this page
does not claim to have invented it. What is new is that the economics
make it worth resolving in a first profile rather than deferring, and
that this is the first page to argue the trade rather than defer it
too. Solana's combination of sub-cent fees,
SPL-USDC, and a transaction-level fee payer makes a small
disbursement economically sensible on-chain — and every consequence
of taking that option lands on the beneficiary's privacy, which is
the one thing this seat exists to protect. That trade is
[its own section](#settlement-is-where-this-binding-gets-hard), and
this page does not pretend it is settled.

## At a glance

A design and feasibility document, not implementation documentation.
Its current conclusions:

- **Boundary:** the notary and eligibility ports bind as on every
  sibling. A **settlement** binding is *proposed but not resolved* —
  and if taken, it is a separate program under a separate profile ID,
  never bolted onto the anchor program.
- **Credentials:** SAS holds the issuer, schema, policy, scope, and
  expiry; **no per-beneficiary attestation is ever written on-chain**,
  because a public attestation is exactly the linkability this seat
  removes.
- **Authorization:** the root-update gate checks the signer against
  the SAS credential's authorized signers — mutable authority
  resolved at call time, which is weaker pinning than BNB's immutable
  admin and buys revocation in exchange.
- **Nullifiers:** a PDA-per-nullifier registry is the proposed pilot
  shape; ZK Compression address trees are the measured candidate, not
  the assumed answer.
- **Proofs:** Groth16 over BN254 through the `alt_bn128` syscalls.
  That is a **third proof system**, not merely a third curve, and it
  brings a per-circuit trusted setup the existing PLONK work does not
  have.
- **Delivery:** nothing is implemented. The build order starts with a
  compute-unit and Poseidon-equivalence measurement against a real
  compiled circuit, and ends with an audited devnet pilot before any
  mainnet-beta decision.

Readers evaluating the architecture should start with
[settlement](#settlement-is-where-this-binding-gets-hard),
[nullifier uniqueness](#nullifier-uniqueness-has-a-baseline-and-a-candidate),
and [the proof-system choice](#groth16-over-bn254-a-third-proof-system-not-a-third-curve).
Implementers should start with [Build order](#build-order) and
[Open profile questions](#open-profile-questions).

## Where the notary boundary ends

Unchanged by the chain. What `UI-Charity.md` §8.1 scopes the notary
port to — campaign status and revision commitments, donation and
disbursement **receipt commitments**, aggregate fund-flow state,
**nullifier uniqueness** per campaign and epoch, and inspectable
policy and status changes — is what the anchor program does, and
nothing else.

Under the notary profile ID the anchor program holds no lamports
beyond the rent its own accounts require, owns no token accounts,
mints nothing, and executes no transfer. An anchored digest proves
*the operator anchored those exact bytes at that time* — never that
money moved or that aid arrived. That refusal is not weakened by the
settlement proposal below: a settlement rail is a separate
`financialBindings` entry with its own program, its own finality,
refund, and reversal mapping, and its own audit. Two programs, two
profile IDs, two declarations in the manifest. Collapsing them into
one program because both happen to run on the same ledger would make
every anchor write a potential fund movement, which is precisely the
authority separation `Charity.md` §1 exists to prevent.

Compliance runs the same way it does on the eUTXO sibling, and stating
it as a positive is clearer than stating it as an absence: screening
obligations — the checks an issuer performs on an organization, and
any identity or sanctions screening performed before an eligibility
credential is issued — are discharged **off-chain, at issuance, by
the party holding that authority**. What reaches the chain is a proof
that a predicate holds, never the material the screening ran on.

## Composing with the Solana Attestation Service

SAS is the reason this binding has less to build than its siblings,
and the mapping onto the contract's objects is close enough to be
worth stating field by field — and precise enough to show where it
stops.

| `Charity.md` object | SAS carrier | What the profile must still supply |
|---|---|---|
| `OrganizationCredential` (§6.2) — issuer, subject, policy, scope, validity, revocation | A Credential account naming the issuing organization and its authorized signers; a Schema account fixing the attested shape | The canonical object and its signature. SAS proves an issuer is registered; it does not carry Onym's canonical bytes, which the receipt digests hash |
| `EligibilityPolicy` (§6.7) — predicate, accepted issuers, proof system, public inputs, nullifier scope | A Schema account, versioned, referenced by campaign state | The predicate, its circuit, its public-input layout, and the nullifier derivation. SAS names a policy; it does not verify one |
| `TrustPolicy` (§6.1) — which issuers and assurance levels a user accepts | Nothing. This stays local to the user's application, as the contract requires | The client-side check that the SAS credential behind a campaign is one the user pinned — issuer trust is never transitive, and "registered in SAS" is not "accepted by this user" |
| Per-beneficiary eligibility attestation | **Deliberately nothing** | The signed beneficiary leaf, delivered to the device over the Onym transport, and its membership proof |

That last row is the design's centre of gravity. SAS supports
per-attestation expiry, and using it for beneficiaries would be the
obvious move — and would publish, next to a wallet address, the fact
that a named issuer attested that subject under an aid policy. That
is a public beneficiary roster in all but name, and `Charity.md` §9.2
forbids one. So beneficiary-level validity is **committed into the
signed leaf and enforced in-circuit** as part of the eligibility
statement, not delegated to SAS. The cost is explicit: expiry becomes
this profile's responsibility, needs its own circuit constraint, and
needs a fixture proving an expired leaf cannot satisfy the predicate.

The second cost of that empty row is larger, and must not hide behind
the word *revocation*. Because nothing per-beneficiary exists
on-chain to revoke, **there is no per-beneficiary revocation**.
Removing a credential's authorized signers stops the tree being
extended; every leaf already inside a posted root stays claimable,
because a membership proof is proved against a root that does not
un-publish. In-leaf expiry is therefore the *only* beneficiary-level
invalidation this design has, which raises the stakes on the
constraint above: an issuer that must withdraw one person's
eligibility before it expires has to post a new root under a policy
whose predicate excludes them, and the profile must state what that
costs a campaign mid-flight. That is an accepted cost of §9.2's
prohibition rather than an oversight — but it is a cost, and it
belongs in the deployment's privacy disclosure, not in a footnote.

The authority chain that results is worth writing out, because a
membership proof is only ever as meaningful as the authority over the
root it is proved against:

1. A SAS Credential names the organization and carries its authorized
   signer set; a SAS Schema fixes the policy and its scope.
2. The issuer signs a beneficiary commitment **off-chain**, under an
   authorized signer key of that credential, and delivers it to the
   device over the Onym transport.
3. An authorized updater posts the resulting eligibility root to the
   campaign account.
4. The program authorizes that root update by checking the updater
   against the credential's authorized signers — so an on-chain root
   is only ever as trusted as the SAS-registered issuer behind it,
   and removing a credential's authorized signers, or pausing its
   schema, revokes the ability to extend the tree.

Step 4 is the profile's most important negative fixture: a signer
removed from the credential can no longer extend the eligibility
tree. Without it, "authorized" is a comment.

## Authorizing an operator write without an immutable admin

The [BNB binding](charity-bnb.md) gates operator-attested writes on
an admin address baked immutably into hashed bytecode. Solana has
signers, so the mechanism is easy — but the choice of *what to check
the signer against* is not, and the two candidates differ in a way
clients feel.

| Gate | How it works | What it costs |
|---|---|---|
| **Admin pubkey fixed at registration**, stored in the campaign account | The instruction asserts that pubkey signed | Simple, and revocation-free — but weaker than it looks, and weaker than the EVM sibling. BNB's admin is `immutable` inside hashed bytecode, so pinning the deployment pins who may write; here the admin lives in **account state**, outside the program-data hash, so a client that pins the program has pinned nothing about it. Rotation is a new campaign registration, or an admin-change instruction if the profile allows one — not a new deployment. The client must read the account, and the profile must say whether that state may change at all |
| **SAS credential lookup** — resolve the credential account and check the signer against its current authorized signers | The instruction reads the credential as an account and compares | Issuer-level revocation works — narrowly, and that is the point of composing with SAS: removing a signer stops *future* root extension. Authority is mutable state outside the program here too, resolved at call time, so the client must read the credential as well and the manifest must declare which one |

The working answer is **both, at different layers**: the campaign's
own administrative writes (registration, revision advance, status
changes) gate on a pubkey fixed at registration, and **root updates**
gate on the SAS credential, because root updates are the writes whose
authority must be revocable. The profile states which instruction
uses which gate, and the client verification story has to carry the
consequence honestly: a deployment's declared powers are only
verifiable if the client checks the credential account as well as the
program.

Proof-authorized writes need no gate, exactly as on the siblings.
Anchoring a claim is signer-agnostic: validity comes from the
eligibility proof checked against the program's own state, never from
who submitted the transaction. The fixture checks the split both ways
— operator instructions refuse other signers, claim anchoring
succeeds from an arbitrary fee payer.

## Nullifier uniqueness has a baseline and a candidate

Solana makes non-membership cheap to express and easy to get subtly
wrong. Two shapes, and the second must not be assumed into the design
before it is measured.

- **A PDA per nullifier.** The program derives an account address
  from the campaign, epoch, and nullifier, and creates it during the
  claim instruction. Creation fails if the account already exists, so
  uniqueness is enforced by the runtime itself rather than by
  program logic — the smallest correct thing that can work.
  Self-contained: no indexer, no external prover, no off-chain
  service between a beneficiary and their claim. It is
  **rent-bearing**, and permanently so.
- **A ZK Compression address tree.** An indexed Merkle tree proving
  non-membership before insertion — precisely the operation
  double-claim prevention needs — with the validity proof checked
  on-chain through the same BN254 syscalls the eligibility verifier
  uses, at a state cost per leaf far below per-account rent. It
  differs from the baseline in proof plumbing, account model,
  concurrent-write behaviour, and, decisively, in **taking a
  dependency on off-chain infrastructure** to produce the validity
  proofs and index the tree.

The conditional answer: **the PDA registry is the pilot shape**, and
compression is promoted to the production path only if a published
measurement of both — compute, state cost, and concurrent-claim
behaviour — says it wins. The reason for that ordering is not cost.
It is that the compressed route inserts an indexer and a proof
service into the path between a beneficiary and their aid, and a seat
whose whole argument is that no single party can withhold help should
adopt that dependency deliberately, with its liveness and censorship
properties stated, rather than inherit it for a rent saving.

Two rules the profile must carry whichever shape wins, both of which
are easy to omit and unrecoverable afterwards:

- **A nullifier account may never be closed.** Solana lets an account
  be closed and its rent reclaimed. Closing a nullifier PDA
  resurrects the entitlement it retired — a spent claim becomes
  claimable again, silently, and the only public trace is a rent
  refund. The program must expose no close instruction for these
  accounts under any authority, and the fixture must attempt it and
  fail. Rent on a nullifier is not a deposit; it is the cost of the
  guarantee.
- **Scope lives in the derivation, not in a lookup.** The nullifier is
  computed in-circuit from the credential secret, campaign, and epoch
  and constrained to the same secret that satisfies the predicate;
  the PDA seeds bind campaign and epoch so that a value valid in one
  scope is a different account in another. Campaign *revision* is
  deliberately not an input — a policy update must not mint a second
  claim in the same window — and the profile names a fixture for
  exactly that.

## Groth16 over BN254: a third proof system, not a third curve

This is the binding's largest new cost, and the page that pretends
otherwise would be the one that gets audited badly.

`onym-contracts` today holds a TurboPLONK prover and verifiers over
**BLS12-381** — the stack the
[Stellar notary runs in production](notary-stellar.md), and the one
the [Cardano plan](charity-cardano.md) expects to reuse. The
[BNB plan](charity-bnb.md) adds **BN254 PLONK**, still unbuilt. A
Solana binding as designed adds **BN254 Groth16**, and Groth16 is not
a curve swap:

- **The curve is shared with BNB, the backend is not.** A BN254
  Groth16 prover in the mobile Rust FFI is new work that no existing
  or planned Onym binding shares. `onym-contracts`' own history records a
  Groth16 path that was dropped; reviving it is a decision, not a
  reuse.
- **Groth16 needs a per-circuit trusted setup.** PLONK's universal
  SRS does not carry over. Every eligibility predicate this binding
  ships requires its own ceremony, with published transcript,
  identified participants, and a verifying key a third party can
  recompute — and a compromised ceremony forges eligibility proofs
  silently, which for this seat means fabricated claims on real aid.
  The proposal this design draws on does not name this cost; the
  profile must, and the build order below funds it as a step.

What Solana gives back is that verification is a **measured
engineering parameter** rather than an open question, which is not
true of every ledger this book profiles. BN254 pairing and G1
arithmetic are native syscalls, and Poseidon is a syscall. Light
Protocol's `groth16-solana` publishes end-to-end benchmarks — plain
Groth16 verification at roughly 78k–109k compute units for one to
eight public inputs, and its BSB22 single-commitment path at roughly
211k–242k over the same range, measured under `mollusk` with
deterministically regenerated proofs and keys. Read those with three
caveats the profile must repeat rather than bury:

1. **They are upstream measurements of a verifier, not of our
   circuit.** The number a reviewer cares about is what *our*
   compiled circuit costs at *our* public-input layout, measured
   under the same methodology so the two are comparable. That
   measurement is build step 1.
2. **Which path applies is a property of how the circuit was
   generated**, together with the public-input count and
   serialisation overhead. The plain path fits inside the default
   per-instruction compute allocation; the BSB22 path requires an
   explicitly raised limit. Both sit far below the per-transaction
   ceiling, so this is a budgeting question, not a feasibility one —
   but the profile must state which path it ships and benchmark
   against that one, not the flattering one.
3. **Cite the repository at a pinned commit.** Published crate
   metadata and secondary sources carry stale figures and stale
   paths; two of them contradict the current tree.

One equivalence has to be proved rather than assumed. The circuit's
Poseidon must match the `sol_poseidon` syscall in field, parameters,
endianness, input framing, and domain separation, or an in-circuit
commitment and its on-chain recomputation disagree and every claim
fails — or, worse, agrees only sometimes. Choosing "Poseidon" on both
sides does not give this; implementations differ, and at least one
widely used standard-library Poseidon is not byte-compatible with the
syscall. A differential test against `sol_poseidon` is an acceptance
criterion, not a nicety.

## Atomicity is free; contention is not

Nullifier consumption and claim anchoring must happen with no window
between the eligibility check and the consumption. One Solana
instruction does the whole sequence — verify the proof, check
campaign status, revision, and epoch, create the nullifier account,
write the anchor — and an instruction either succeeds entirely or
reverts entirely. The property the EVM sibling gets by doing both
writes in one call, this gets the same way.

Contention is more interesting, and the tempting version of this
paragraph is wrong. The claim-specific accounts are disjoint — the
campaign account read-only, distinct nullifier accounts — so Sealevel
schedules *those* concurrently. But every transaction has a **fee
payer, and the fee payer is writable**; under the relayer model below
one relayer key pays for every claim in a campaign and funds each
nullifier account's rent, so a single-key relayer serialises claims
on its own account against the per-account write budget a block
allows. **Contention therefore exists in notary-only mode**, before
any settlement rail is considered, and a page that located it only at
the vault would be describing a different deployment than the one it
proposes.

The mitigation is key count, and each count shards only its own
account: **multiple relayer fee-payer keys** shard fee-payer
contention, and — if the settlement binding is taken — **multiple
vault sub-accounts** shard vault contention, because every SPL-USDC
payout debits its source token account and a single-vault campaign
serialises on that one too. Two contended accounts, two independent
shard counts, and throughput bounded by whichever is scarcer.
Solana's contribution is that both are legible and shardable rather
than implicit — and the profile requires them *measured* rather than
asserted, because "shardable" is a design claim until a number
exists.

**Sealevel is not a privacy primitive.** The concurrency it does give
removes a queue whose ordering would otherwise amplify timing
correlation between claims, which is a narrow but real benefit;
inclusion slots, signatures, and account accesses stay observable,
and this binding's unlinkability rests on the credential and
nullifier design, not on the scheduler. A fee payer shared across
every claim cuts the other way, and it is the EVM sibling's
[single gas-paying submitter](charity-adversary.md#5-the-single-gas-paying-submitter)
row with a second job — it pays *and* it is the account every claim
contends on.

## Paying fees for someone who holds no SOL

A beneficiary holds no SOL and should not have to. Solana's
transaction format designates a fee payer, so any account can pay for
another with no contract to deploy: the Onym transport carries the
proof to a relayer, and the relayer signs as fee payer. Kora, the
Foundation's fee-relayer infrastructure, is the candidate
implementation, and the model is the one the
[Stellar notary already runs](notary-stellar.md): the operator's
account signs and pays, no end-user chain key ever exists, and the
relayer's "ok" is transport progress — the client reconciles against
the chain before believing anything.

What that costs, stated where a beneficiary can read it: **the
relayer sees the transaction it signs**, including the destination it
pays to. It is also a censorship point — a relayer that declines to
submit a claim is a party that can withhold aid without any on-chain
trace. Multiple fee-payer keys shard throughput; they do not remove
the trust. The mitigation the contract already implies is that claim
anchoring is signer-agnostic, so a beneficiary who *can* pay fees, or
who finds any other submitter, is never locked out by one relayer's
refusal. The profile must keep that path open and the UI must not
hide it.

## Settlement is where this binding gets hard

Everything above is a notary binding with better credential
plumbing. This section is the part with no sibling precedent, and it
is not resolved here.

`Charity.md` §6.8 is explicit that public state may contain only the
claim digest, the scoped nullifier, and a randomized, claim-scoped
recipient commitment, and that the payout coordinate stays sealed to
the named delivery provider. An SPL-USDC transfer in the same
**transaction** as the claim anchor **publishes the destination token
account in that transaction's account list**, permanently joined to
the nullifier that retired the entitlement. Splitting the payout into
its own *instruction* changes nothing — the account list is
transaction-scoped, so the two appear together either way — which is
what the third option below is actually doing. The proof hid which leaf
the claimant was; the settlement then publishes an address that
receives the money, and every downstream movement of that balance is
chain-analysable by anyone.

The precise formulation, which the UI must use with beneficiaries and
not only with reviewers: this is **identity unlinkability, not
transaction confidentiality**. Amounts and destinations are public,
as SPL settlement requires. What the protocol never publishes is a
link between a credentialed person and that receiving address.

Three ways to take the option, with what each costs:

- **Settle through the delivery provider.** The program anchors, the
  named financial provider disburses under its own legal authority —
  the shape all three siblings take. Preserves the §6.8 boundary
  exactly, and gives up the fee and speed argument that made Solana
  settlement attractive.
- **Settle on-chain to a fresh recipient account per claim**,
  rent-funded by the relayer, its address never reused. Preserves
  unlinkability *between claims*, keeps this claim's payout public,
  and pushes the exposure onto whatever the beneficiary does next —
  which is exactly where a beneficiary has the least support.
- **Settle on-chain in a transaction separated from the anchor**, on
  a declared schedule or in batches, so the join is weakened by
  timing rather than published outright. Weakens the audit trail the
  claim→disbursement join exists to provide, and inherits the
  anchor-batching analysis the BNB profile still lists as open.

The profile picks one, states it in the deployment's privacy
disclosure, and the UI discloses it **before the claimant signs**, per
`Charity.md` §11's requirement that a possible public transaction
graph be shown before authorization. Confidential amounts through
Token-2022 are out of scope for a first version: the proof program
behind them has had a feature-gate history this seat should not
depend on, and shielding the recipient is this design's job in any
case.

And the non-custody rule survives all three. `Charity.md` §11 makes
the abstract boundary non-custodial: it requests quotes, obtains
authorization, and verifies outcomes. A program-owned vault holding
campaign funds is custody by any reading a regulator would apply, so
a settlement binding does not make the *protocol* custodial — it
makes some named legal party custodial, and the deployment must say
which, with the donation state machine's `refund-pending`,
`refunded`, and `reversed` states given concrete meaning on a ledger
that has no reversal primitive. An irreversible transfer cannot
implement `reversed` by itself; what implements it is a party with an
obligation. Naming that party is a precondition of declaring the
binding, not a follow-up.

## Every surface that can carry bytes

The EVM fixture greps emitted logs and written storage slots. Solana
has more surfaces, and one of them — program logs — is the easiest
place in this entire design to leak a name during debugging.

| Surface | Carries | Rule under this profile ID |
|---|---|---|
| Account data | Campaign state, anchors, roots, nullifier markers | Typed commitments, digests, scoped nullifiers, statuses, timestamps only. No free-text field, no variable-length blob without a declared schema |
| Instruction data | Proof bytes, public inputs, operation arguments | Same discipline. The sealed recipient payload never appears in instruction data at all |
| Program logs (`msg!`) | Anything a developer prints | Structured, enumerated events only. No proof diagnostics, no input echo, no error strings carrying argument values |
| Account addresses and PDA seeds | Campaign, epoch, nullifier | Seeds derived from public campaign data and the scoped nullifier only — never from anything credential-linked, or the address becomes the identifier the nullifier was designed not to be |
| Memo instruction | Arbitrary UTF-8 alongside a transfer | **Prohibited.** A conforming transaction carries no memo |
| Token accounts (settlement binding only) | Owner, mint, balance | Public by construction; governed by the settlement section above, not by this table |

The negative fixture set plants names, IBANs, emails, and addresses
in the input objects and greps **every account this deployment
writes, all instruction data, and the full log output of every
instruction** — zero hits to pass — with a separate assertion that
the sealed recipient payload appears nowhere in instruction data, and
that no transaction carries a memo. None of these fixtures exist;
"requires" is the strongest true verb on this page.

What a Solana explorer adversary sees differs from the BSC rows of
the [adversary's view](charity-adversary.md) in three ways. Program
logs are indexed and rendered by default, which makes a leaked string
*more* discoverable than an EVM storage slot. Fee payers are
first-class in every explorer view, so the relayer's key set is a
persistent, linkable fingerprint on every claim. And if the
settlement binding is taken, token-transfer indexing gives a
follow-the-money view no sibling offers — for free, to anyone.
Small-count correlation is unchanged and remains the sharpest edge.

## Deployment identity, upgrade authority, and finality

**Identity.** The analogue of chain ID plus address plus runtime code
hash is **cluster genesis hash + program ID + the hash of the
deployed program data**, all three verified before first use.
Mainnet-beta and devnet are in scope; testnet is not, as opBNB is not
for the [EVM sibling](charity-bnb.md).

**Upgradeability is the trap.** Solana programs deploy upgradeable by
default: an upgrade authority can replace the executable at the same
program ID, so a client pinning a program ID pins nothing about the
code that will run tomorrow. The other three bindings prohibit
proxies and upgrade patterns outright, and this one must reach the
same place through a different mechanism — the profile requires the
upgrade authority to be **revoked**, making the program immutable,
before a deployment may be declared, with the client verifying the
authority is `None` rather than trusting a declaration that it is.
A deployment whose authority is retained is a different security
model and, if it is ever permitted, must be declared as one and
pinned to a named multisig with its members published. Verifiable
builds should map the on-chain program hash back to the source a
reviewer read; without that, the hash proves only that the bytes did
not change, not what they do.

**Finality.** Commitment levels are client-visible states, not
implementation detail. `processed` and `confirmed` are progress;
nothing may be shown as final before reconciliation at `finalized`,
which under current consensus is a matter of seconds rather than the
sub-second numbers marketing materials quote. A confirmed anchor that
does not survive to `finalized` is a `conflicting_state` **security
event, not a retry** — for a claim write, the rebuild path terminates
in either an idempotent identical anchor or a scoped
already-claimed refusal, both correct, and the event is still
reported. Faster finality is a roadmap item, not a correctness
dependency: the design is correct at today's numbers.

## Errors and their retry semantics

Anchor programs return custom error codes in the transaction result,
so this binding keeps the BNB shape — a class decided **from the
chain itself** — rather than the Cardano one, where classification
moves off-chain. One class is new, and it is the most common failure
in practice.

| Class | Condition | Client behavior |
|---|---|---|
| refresh-and-rebuild | Stale campaign revision; epoch not current | Re-resolve, re-consent to the new revision, rebuild the presentation (a new epoch derives a new nullifier), resubmit |
| retry-as-transport | **Blockhash expired**, or the transaction was dropped before inclusion | Solana-specific and routine: the transaction never executed, nothing was consumed, and resubmitting the same operation is correct. A durable nonce is the profile's option for slow provers on poor connections |
| terminal scoped refusal | The nullifier account already exists in this campaign and epoch | The entitlement was already claimed in this scope. Show the scoped refusal; never a person identifier, never a retry |
| refuse-as-defect | Invalid proof, out-of-field value, unknown policy, unauthorized signer | Generator or tooling bug; retrying cannot help. Proof diagnostics stay private and out of the logs |
| security event | An anchor contradicting a previously observed anchor; a confirmed anchor that does not reach `finalized` | Preserve evidence, raise the incident path, never silently resubmit or overwrite |

Compute-unit exhaustion deserves naming because it looks like a
transient failure and is not: a claim that exceeds the requested
limit fails deterministically and will fail identically on retry
until the limit is raised. It belongs in refuse-as-defect, and the
profile fixes the requested limit per instruction rather than leaving
clients to guess.

## The operator manifest

The relayer's signed, byte-served operator manifest — live today on
the [Stellar notary side](notary-stellar.md#the-live-operator-manifest)
— would gain the charity profile entry, the anchor program
deployments it administers, and Solana network entries binding the
Ed25519 operator identity to the fee-payer accounts that pay for
submission and the admin account whose signature the operator
instructions accept. Solana accounts are Ed25519 too, which removes
the cross-key-type awkwardness the EVM entries carry — and the
profile must still refuse to conflate them: the operator's manifest
identity and its on-chain fee payer are different capabilities that
happen to share a signature scheme.

The normative client check carries over unchanged and gains one row.
Deployment verification MUST compare the manifest's declared admin
against the campaign account's stored admin, the declared program
data hash against the chain, **the upgrade authority against `None`**,
and — where root updates gate on SAS — the declared credential
against the one the program actually reads. Without those
comparisons, "declared powers match program-enforced reality" is not
verifiable. A deployment absent from the manifest does not exist for
clients, whatever is on the chain.

## Honest status

- **Nothing on this page runs, and nothing specifies it.** There is
  no `UI-Charity-Solana.md`: `onym-system` holds exactly
  `Charity.md`, `UI-Charity.md`, and `UI-Charity-BNB.md`. There are
  no charity programs, no charity circuits on any curve, and no
  Solana endpoints in the relayer — neither
  [`onym-contracts`](https://github.com/onymchat/onym-contracts) nor
  the relayer repository contains the string `solana` or `charity` in
  any source file. *(Repository state verified 21 August 2026.)*
- **Nothing is reused.** This is the honest difference from the
  Cardano plan, which starts with a working prover. The BN254
  Groth16 prover backend, the circuits, the programs, the trusted
  setup, and the relayer's Solana backend are all new, and none of
  them is shared with another Onym binding today. The BN254 curve is
  shared with the [BNB plan](charity-bnb.md); the proof system is
  not, so the backends are separate builds.
- **What is reused is external, and that is the argument.** SAS
  supplies the credential layer, `groth16-solana` the verifier, Kora
  the fee relaying, and — if measurement supports it — ZK Compression
  the nullifier set. Each is a dependency on a party outside this
  project, and the profile names each one with its version and its
  failure mode rather than treating it as infrastructure.
- **Network-roadmap items are design input, not commitments.**
  Syscalls for BN254 G2 and BLS12-381 would let this binding converge
  on the curve the Stellar and Cardano work already use, collapsing
  three curve stacks toward one; larger transactions and reduced rent
  would cut per-claim state cost. Nothing in this design requires
  any of them, and their activation status must be re-verified when
  the profile is written rather than inherited from this page.
- The abstract contracts this would answer to (`Charity.md`,
  `UI-Charity.md`) are merged drafts (0.1, August 2026), and the
  `Charity.md` §6.8 wording question the BNB profile flags upstream —
  campaign-scoped fields in public claim anchors — is unresolved and
  binds this profile too. This page also raises a second §6.8
  question the siblings never had to:
  [what a settlement binding may publish](#settlement-is-where-this-binding-gets-hard).
- The design input behind this page is an August 2026 funding
  application for the Solana implementation. Like the July 2026
  memorandum `UI-Charity.md` §8.3 declines to treat as normative, it
  is design input — not a protocol dependency, and not evidence that
  anything here is funded, adopted, or built.

## Build order

Solana proceeds on its own dependency chain. No sibling gates it and
it gates none, which is the house position rather than a new claim:

1. **Measure before specifying.** Two results can invalidate the
   profile's proof section, and both are cheap: a compute-unit
   benchmark of the *compiled* eligibility circuit at its actual
   public-input layout, reported against the corresponding upstream
   figure under the same methodology and naming which verifier path
   it needs; and a differential test showing in-circuit Poseidon
   commitments reproduce `sol_poseidon` byte for byte. The same step
   publishes the PDA-versus-compressed nullifier comparison, because
   the shape fixes the account model the programs are written
   against.
2. **Profile, in two halves** — `charity/UI-Charity-Solana.md`. The
   proof-independent half can be written in parallel with step 1:
   account layouts and their public fields, the two authorization
   gates, SAS credential and schema binding, deployment identity and
   the upgrade-authority rule, the error taxonomy, and the extended
   negative-PII fixture set over the five surfaces this profile ID
   governs. The
   proof half — public-input layout, proof encoding, requested
   compute limits — waits on step 1. `UI-Charity.md` §8.3 sets the
   bar and applies unchanged: until the document exists and passes
   conformance tests, "uses Solana" is an implementation choice, not
   evidence that the boundary is satisfied.
3. **Circuits and ceremony** — the eligibility constraint system over
   BN254, including in-circuit expiry, and the per-circuit Groth16
   trusted setup with a published transcript and identified
   participants. The ceremony is a deliverable with a named owner,
   not a build detail; a verifying key nobody can audit the origin of
   is a forgery surface for claims on real aid.
4. **Prover** — the BN254 Groth16 backend in the mobile Rust FFI.
   New work, shared with nothing, and the step that has to hold the
   Poseidon parameterisation fixed against step 1's differential
   test.
5. **Programs** — the anchor program plus the verifier, deployed with
   the upgrade authority revoked and a verifiable build published.
6. **Audit** — an independent review of the circuits, the verifier,
   and the authority model, scoped to whichever verifier path ships
   rather than assuming upstream coverage transfers. A line item, not
   an assumption.
7. **Relayer Solana backend** — transaction building, fee-payer key
   management, blockhash and durable-nonce handling, commitment-level
   reconciliation, and fork watching. New; no other binding shares it
   today.
8. **Declare, list, prove** — manifest entries, discovery listing,
   and the fixture suite green, in that order, before any real
   campaign binds this deployment.
9. **Devnet pilot, then a mainnet-beta decision** — real campaigns end
   to end on devnet, conformance vectors green, and only then a
   mainnet decision, matching the [Stellar](charity-stellar.md) and
   [Cardano](charity-cardano.md) plans' final phase. The ordering is
   the point: the mainnet decision follows the audit, because the
   alternative is asking beneficiaries to be the test surface for an
   unaudited verifier.

The settlement question is deliberately not a numbered step. It is
resolved at step 2 as a scope decision — notary-only, or notary plus
a separate settlement profile — and if the second is chosen, that
profile gets its own build chain, its own audit, and its own named
legal counterparty.

## Open profile questions

Collected, because each is a place this page refused to invent an
answer:

1. **Settlement scope** — notary-only like all three siblings, or a
   separate on-chain settlement profile; and if the latter, which of
   the three publication shapes, and who is the custodial legal
   counterparty implementing `refund-pending`, `refunded`, and
   `reversed` on a ledger with no reversal primitive.
2. **Nullifier set shape** — the PDA registry is the pilot answer;
   what is open is whether the measured compute, state, and
   concurrency profile of ZK Compression address trees justifies
   taking an indexer and proof-service dependency into the aid path.
3. **Verifier path and its budget** — plain Groth16 or the BSB22
   commitment path, decided by what the compiled circuit requires,
   with the requested compute limit fixed in the profile rather than
   chosen per client.
4. **Trusted-setup governance** — who runs the ceremony per circuit,
   who participates, where the transcript is published, and what
   happens to deployed campaigns when a circuit is revised.
5. **Root-update authority** — SAS credential lookup at call time
   versus a pinned signer set, and how a client verifies the mutable
   half without a second network round-trip it cannot afford.
6. **Fee-payer concentration** — how many relayer keys, declared or
   not, and whether their reuse across campaigns widens the existing
   [gas-paying-submitter row](charity-adversary.md#5-the-single-gas-paying-submitter)
   enough to need its own. The count is also a throughput parameter,
   so privacy and liveness pull on the same dial here.
7. **Anchor batching** — open on BNB, open here, and sharper if
   settlement is on-chain, because batching then trades audit
   granularity against timing correlation on the money as well as on
   the claim.
8. **Statement separation from BNB** — a shared curve means
   separation rests on the statement tag and profile ID rather than
   on the curve, and the cross-rejection fixtures must be designed
   for that, not inherited from a cross-curve pair. The differing
   proof systems make an accidental cross-acceptance unlikely; the
   fixture proves it rather than arguing it.

## Next steps

- [Charity](charity.md) — the abstract seat this would bind.
- [BNB Chain](charity-bnb.md) — the merged specification whose
  section shape this page mirrors, and whose curve it shares without
  sharing its proof system.
- [Cardano](charity-cardano.md) — the sibling that reaches the same
  obligations on a ledger with no sender, and whose prover this
  binding cannot reuse.
- [Stellar/Soroban](charity-stellar.md) — the planned reference
  binding, and the relayer and manifest discipline this plan reuses.
- [The adversary's view](charity-adversary.md) — what a public trail
  exposes; the log, fee-payer, and settlement rows above are the
  Solana-specific deltas.
- [Who holds which role](charity-roles.md) — the abstract roles
  mapped to concrete parties, including the unassigned ones.
