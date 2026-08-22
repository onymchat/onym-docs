# Charity — Solana

*Seat implementation page, draft 0.1 — 21 August 2026.*

**Status:** Plan; profile and code not implemented. See
[Honest status](#honest-status).

**Profile:** must be written as `charity/UI-Charity-Solana.md` in
`onym-system`. It must bind the
[`charity/Charity.md`](https://github.com/onymchat/onym-system/blob/main/charity/Charity.md)
boundary's **notary and eligibility bindings** to Solana. Mainnet-beta
and devnet are in scope; testnet is not. It also proposes the first
**financial binding** described by a charity page. **Code:** none.

Solana differs from its three siblings for two reasons. First, the **Solana
Attestation Service** (SAS) already provides much of the credential layer
that other bindings must build. This Foundation-maintained public good has
Credential, Schema, and Attestation accounts for authorized issuer signers,
versioned schemas, and per-attestation expiry. Those fields closely match the
organization credential and eligibility policy required by `Charity.md` §6.2
and §6.7.

Second, Solana is the first candidate where **settlement on the same ledger is
worth designing now**. The [Stellar plan](charity-stellar.md) already names a
later, separate USDC `financialBindings` profile with the same refund and
reversal precondition. Solana's sub-cent fees, SPL-USDC, and transaction-level
fee payer make small on-chain disbursements economically sensible. The privacy
cost falls on the beneficiary. The trade is [examined below](#settlement-is-where-this-binding-gets-hard)
and remains unresolved.

## At a glance

This is a design and feasibility document.

- **Boundary:** notary and eligibility bind as on every sibling; settlement is *proposed but not resolved* as a separate program and profile ID.
- **Credentials:** SAS holds issuer, schema, policy, scope, and expiry; **no per-beneficiary attestation is ever written on-chain**.
- **Authorization:** root updates use SAS authorized signers, with mutable authority resolved at call time and weaker pinning than BNB's immutable admin.
- **Nullifiers:** a PDA-per-nullifier registry is the proposed pilot; ZK Compression address trees are a candidate to measure.
- **Proofs:** Groth16 over BN254 through `alt_bn128` is a third proof system and requires a per-circuit trusted setup.
- **Delivery:** nothing is implemented; measurement comes first, followed by an audited devnet pilot before any mainnet-beta decision.

Architecture review should start with
[settlement](#settlement-is-where-this-binding-gets-hard),
[nullifier uniqueness](#nullifier-uniqueness-has-a-baseline-and-a-candidate),
and [the proof-system choice](#groth16-over-bn254-a-third-proof-system-not-a-third-curve).
Implementation starts with [Build order](#build-order) and
[Open profile questions](#open-profile-questions).

## Where the notary boundary ends

The chain does not change the notary boundary. Under `UI-Charity.md` §8.1, the
anchor program handles only campaign status and revision commitments, donation
and disbursement **receipt commitments**, aggregate fund-flow state,
**nullifier uniqueness** per campaign and epoch, and inspectable policy and
status changes.

Under the notary profile ID, the anchor program holds only the lamports needed
to rent its accounts. It owns no token accounts, mints nothing, and executes no
transfer. An anchored digest proves *the operator anchored those exact bytes at
that time*. It never proves that money moved or aid arrived.

Any settlement rail is a separate `financialBindings` entry with its own
program, profile ID, finality, refund and reversal mapping, audit, and manifest
declaration: two programs, two profile IDs, and two declarations. Combining
them would make every anchor write a potential fund movement. That would
violate the authority separation in `Charity.md` §1.

Screening obligations are discharged **off-chain, at issuance, by the party
holding that authority**. This includes checks on an organization and any
identity or sanctions screening before issuing an eligibility credential. The
chain receives only proof that a predicate holds, never the screening material.

## Composing with the Solana Attestation Service

SAS supplies part of the credential layer. The profile must supply the rest.

| `Charity.md` object | SAS carrier | What the profile must still supply |
|---|---|---|
| `OrganizationCredential` (§6.2) — issuer, subject, policy, scope, validity, revocation | A Credential account naming the issuing organization and its authorized signers; a Schema account fixing the attested shape | The canonical object and its signature. SAS proves registration, but does not carry the canonical Onym bytes hashed by receipt digests |
| `EligibilityPolicy` (§6.7) — predicate, accepted issuers, proof system, public inputs, nullifier scope | A versioned Schema account referenced by campaign state | The predicate, circuit, public-input layout, and nullifier derivation. SAS names a policy; it does not verify one |
| `TrustPolicy` (§6.1) — accepted issuers and assurance levels | Nothing; this remains local to the user's application | A client check that the user pinned the campaign's SAS credential. Issuer trust is never transitive; “registered in SAS” does not mean “accepted by this user” |
| Per-beneficiary eligibility attestation | **Deliberately nothing** | The signed beneficiary leaf, delivered to the device over the Onym transport, and its membership proof |

Beneficiary validity is **committed into the signed leaf and enforced
in-circuit**. A SAS beneficiary attestation would publish, beside a wallet
address, that a named issuer attested the subject under an aid policy. That is
effectively a public beneficiary roster, forbidden by `Charity.md` §9.2.
Expiry therefore needs a circuit constraint and a fixture proving that an
expired leaf cannot satisfy the predicate.

There is also **no per-beneficiary revocation**. Nothing per beneficiary exists
on-chain to revoke. Removing a credential's authorized signers stops future
tree extension, but every leaf in a posted root remains claimable. The root does
not un-publish.

In-leaf expiry is the only beneficiary-level invalidation. To withdraw one
person's eligibility earlier, an issuer must post a new root under a policy
whose predicate excludes them. The profile must state the mid-campaign cost.
This accepted cost of §9.2's prohibition belongs in the deployment privacy
disclosure.

The authority chain is:

1. A SAS Credential names the organization and its authorized signers; a SAS
   Schema fixes policy and scope.
2. The issuer signs a beneficiary commitment **off-chain** with an authorized
   signer key and delivers it over the Onym transport.
3. An authorized updater posts the resulting eligibility root to the campaign
   account.
4. The program checks the updater against the credential's current authorized
   signers. Removing a signer or pausing the schema stops tree extension.

A membership proof is only as meaningful as the authority over its root. The
profile's key negative fixture removes a signer and confirms that it can no
longer extend the tree.

## Authorizing an operator write without an immutable admin

The working answer uses two gates at different layers.

| Gate | How it works | What it costs |
|---|---|---|
| **Admin pubkey fixed at registration**, stored in the campaign account | The instruction asserts that the pubkey signed | Simple and revocation-free, but weaker than the [BNB binding](charity-bnb.md). BNB's admin is `immutable` in hashed bytecode. Solana's admin is account state outside the program-data hash, so pinning the program does not pin the admin. Rotation requires new campaign registration, or an admin-change instruction if allowed. The client must read the account, and the profile must define whether it can change |
| **SAS credential lookup** against current authorized signers | The instruction reads the credential account and compares the signer | Removing a signer stops future root extension. Authority remains mutable state outside the program and is resolved at call time. The client must read the credential, and the manifest must declare it |

Campaign administration, including registration, revision advance, and status
changes, gates on a pubkey fixed at registration. **Root updates** gate on the
SAS credential because their authority must be revocable. The profile states
the gate for every instruction. Clients must check the credential account and
the program to verify declared powers.

Proof-authorized writes need no gate. Claim validity comes from the eligibility
proof checked against program state, not the submitter. Fixtures must show that
operator instructions reject other signers while an arbitrary fee payer can
anchor a claim.

## Nullifier uniqueness has a baseline and a candidate

The PDA registry is the pilot shape. ZK Compression becomes the production
path only if published measurements justify it.

| Shape | Mechanism | Cost |
|---|---|---|
| **One PDA per nullifier** | Derive an account from campaign, epoch, and nullifier; create it during the claim. Existing accounts make creation fail, so the runtime enforces uniqueness | Self-contained, with no indexer, external prover, or off-chain service in the claim path. Every account bears rent permanently |
| **ZK Compression address tree** | Prove non-membership in an indexed Merkle tree before insertion; verify the validity proof through the same BN254 syscalls as the eligibility verifier | Much lower state cost per leaf, but different proof plumbing, account semantics, and concurrent-write behaviour. It depends on off-chain proof production and indexing |

The comparison must publish compute, state cost, and concurrent-claim
behaviour. Compression adds an indexer and proof service between a beneficiary
and aid. Its liveness and censorship properties must be stated before accepting
that dependency for rent savings.

Two rules apply to either shape:

- **A nullifier account may never be closed.** Closing a PDA and reclaiming rent
  silently resurrects its entitlement; the only public trace is a rent refund.
  The program must expose no close instruction under any authority, and a
  fixture must attempt closure and fail. Rent is the cost of the guarantee, not
  a deposit.
- **Scope lives in the derivation.** The circuit derives the nullifier from the
  credential secret, campaign, and epoch, constrained to the secret satisfying
  the predicate. PDA seeds also bind campaign and epoch. Revision is
  deliberately excluded, so a policy update cannot mint another claim in the
  same window. A fixture must cover this case.

## Groth16 over BN254: a third proof system, not a third curve

This binding adds **BN254 Groth16**, a new backend with a per-circuit trusted
setup.

`onym-contracts` has a TurboPLONK prover and verifiers over **BLS12-381**. The
[Stellar notary runs that stack in production](notary-stellar.md), and the
[Cardano plan](charity-cardano.md) expects to reuse it. The unbuilt
[BNB plan](charity-bnb.md) adds **BN254 PLONK**.

- **Only the curve is shared with BNB.** The mobile Rust FFI needs a new BN254
  Groth16 prover. No existing or planned Onym binding shares it.
  `onym-contracts` previously dropped a Groth16 path; reviving it is not reuse.
- **Groth16 needs a per-circuit trusted setup.** PLONK's universal SRS does not
  carry over. Every eligibility predicate needs a ceremony, published
  transcript, identified participants, and a verifying key that a third party
  can recompute. A compromised ceremony can silently forge eligibility proofs
  and fabricate claims on real aid. The source proposal does not name this
  cost; the profile and build plan must.

Solana makes verification measurable. BN254 pairing and G1 arithmetic are
native syscalls, as is Poseidon. Light Protocol's `groth16-solana` end-to-end
benchmarks report about 78k–109k compute units for plain Groth16 with one to
eight public inputs. Its BSB22 single-commitment path reports about 211k–242k
over the same range. These benchmarks use `mollusk` with deterministically
regenerated proofs and keys.

The profile must repeat three caveats:

1. **They are upstream measurements of a verifier, not of our circuit.** Build
   step 1 measures our compiled circuit and actual public-input layout with the
   same methodology.
2. **The applicable path depends on circuit generation**, public-input count,
   and serialisation overhead. The plain path fits the default per-instruction
   compute allocation. BSB22 requires an explicitly raised limit. Both are far
   below the per-transaction ceiling. The profile must name and benchmark the
   shipped path.
3. **Cite the repository at a pinned commit.** Published crate metadata and
   secondary sources contain stale figures and paths; two contradict the
   current tree.

Poseidon equivalence must also be proved. The circuit and `sol_poseidon` must
match in field, parameters, endianness, input framing, and domain separation.
Otherwise on-chain recomputation may reject every claim or agree only
sometimes. At least one widely used standard-library implementation is not
byte-compatible with the syscall. A differential test against `sol_poseidon`
is an acceptance criterion.

## Atomicity is free; contention is not

One instruction must verify eligibility, check campaign status, revision, and
epoch, create the nullifier account, and write the anchor. It either completes
or reverts. There is no window between proof verification and nullifier
consumption.

Contention exists in notary-only mode. Claim-specific nullifier accounts are
disjoint and the campaign account is read-only, so Sealevel can schedule them
concurrently. Every transaction still has a writable **fee payer**. Under the
relayer model, one key pays every campaign claim and each nullifier account's
rent. A single key therefore serialises claims against the block's per-account
write budget.

**Multiple relayer fee-payer keys** shard that contention. If settlement is
chosen, **multiple vault sub-accounts** independently shard contention on the
SPL-USDC source token account. Throughput is bounded by the scarcer shard
count. These are two contended accounts with two independent shard counts. Both
require measurement; shardability alone supplies no number.

**Sealevel is not a privacy primitive.** Concurrency removes a queue that could
amplify timing correlation. Inclusion slots, signatures, and account accesses
remain observable. Unlinkability comes from credential and nullifier design.
A shared fee payer creates the EVM sibling's
[single gas-paying submitter](charity-adversary.md#5-the-single-gas-paying-submitter)
with a second role: it also becomes the account on which every claim contends.

## Paying fees for someone who holds no SOL

A beneficiary needs no SOL or end-user chain key. The Onym transport sends the
proof to a relayer, which signs as transaction fee payer. Kora, the Foundation's
fee-relayer infrastructure, is the candidate implementation. The
[Stellar notary](notary-stellar.md) uses the same model: the operator signs and
pays, its “ok” means transport progress, and the client reconciles against the
chain.

The **relayer sees the transaction it signs**, including the payment
destination. It can also censor. A refused claim leaves no on-chain trace, so
the relayer can withhold aid. Multiple fee-payer keys improve throughput but do
not remove that trust.

Claim anchoring must remain signer-agnostic. A beneficiary who can pay, or can
find another submitter, must not be locked out by one relayer. The profile must
preserve this path, and the UI must expose it.

## Settlement is where this binding gets hard

On-chain settlement has no sibling precedent and remains unresolved because it
publishes the destination.

`Charity.md` §6.8 allows public state to contain only the claim digest, scoped
nullifier, and randomized claim-scoped recipient commitment. The payout
coordinate remains sealed to the named delivery provider. An SPL-USDC transfer
in the same **transaction** as the anchor permanently joins its destination
token account to the nullifier. A separate instruction in that transaction
does not help because account lists are transaction-scoped.

The eligibility proof hides the claimant's leaf. Settlement exposes the
receiving address, amount, and every chain-analysable downstream movement. The
UI must describe this as **identity unlinkability, not transaction confidentiality**.
SPL settlement makes amounts and destinations public. The protocol never
publishes a link between a credentialed person and the address.

| Option | Privacy and operational cost |
|---|---|
| **Delivery-provider settlement** | The program anchors and the named financial provider disburses under its legal authority. This preserves §6.8 exactly but gives up Solana's fee and speed argument |
| **Fresh recipient account per on-chain claim**, funded for rent by the relayer and never reused | Preserves unlinkability between claims. This claim's payout and the beneficiary's next movement remain public, precisely where support is weakest |
| **Separate settlement transaction**, scheduled or batched | Weakens the timing join instead of publishing it directly. It also weakens claim-to-disbursement auditability and inherits BNB's open anchor-batching analysis |

The profile must choose one and include it in the deployment privacy
disclosure. Under `Charity.md` §11, the UI discloses the possible public
transaction graph **before the claimant signs**.

Confidential amounts through Token-2022 are out of scope for the first version.
Its proof program has a feature-gate history on which this seat should not
depend. Shielding the recipient remains this design's responsibility.

All three options preserve the abstract boundary's non-custody rule.
`Charity.md` §11 limits it to requesting quotes, obtaining authorization, and
verifying outcomes. A program-owned campaign vault is custody. A settlement
binding therefore makes a named legal party custodial, not the protocol.

The deployment must name that party. It must also map `refund-pending`,
`refunded`, and `reversed` to a ledger without a reversal primitive. An
irreversible transfer cannot implement `reversed`; a party with an obligation
does. Naming that party is a precondition for declaring the binding.

## Every surface that can carry bytes

The profile governs every public byte surface.

| Surface | Carries | Rule under this profile ID |
|---|---|---|
| Account data | Campaign state, anchors, roots, nullifier markers | Typed commitments, digests, scoped nullifiers, statuses, and timestamps only. No free text or undeclared variable-length blob |
| Instruction data | Proof bytes, public inputs, operation arguments | The same discipline; the sealed recipient payload never appears here |
| Program logs (`msg!`) | Anything printed by a developer | Structured, enumerated events only. No proof diagnostics, input echo, or error strings containing argument values |
| Account addresses and PDA seeds | Campaign, epoch, nullifier | Derive only from public campaign data and the scoped nullifier, never credential-linked data |
| Memo instruction | Arbitrary UTF-8 beside a transfer | **Prohibited.** A conforming transaction has no memo |
| Token accounts, settlement only | Owner, mint, balance | Public by construction and governed by the settlement section |

Negative fixtures plant names, IBANs, emails, and addresses in input objects.
They grep every written account, all instruction data, and every instruction's
full log output. Passing requires zero hits. Separate assertions require that
the sealed recipient payload never enters instruction data and no transaction
contains a memo. These fixtures do not exist; “requires” is the strongest true
verb.

A Solana explorer differs from the BSC rows in the
[adversary's view](charity-adversary.md) in three ways. Indexed, default-visible
program logs make leaked strings easier to discover than EVM storage slots.
First-class fee payers make the relayer key set a persistent fingerprint. If
settlement is chosen, token-transfer indexing gives everyone a follow-the-money
view that no sibling offers. Small-count correlation remains the sharpest edge.

## Deployment identity, upgrade authority, and finality

**Identity.** Clients verify **cluster genesis hash + program ID + deployed
program-data hash** before first use. Mainnet-beta and devnet are in scope;
testnet is not, matching opBNB's exclusion from the
[EVM sibling](charity-bnb.md).

**Upgradeability.** Solana programs are upgradeable by default. An upgrade
authority can replace code behind the same program ID. The other three bindings
prohibit proxies and upgrade patterns outright. Solana reaches the same
immutability by revoking the authority before declaration. Clients must verify
that it is `None`.

Retained authority is a different security model. If ever permitted, it must
be declared and pinned to a named multisig whose members are published.
Verifiable builds should map the on-chain program hash to reviewed source.
Without that mapping, the hash proves only that bytes did not change.

**Finality.** `processed` and `confirmed` are progress states. Nothing is final
before reconciliation at `finalized`, which under current consensus takes
seconds, not the sub-second figures in marketing material. A confirmed anchor
that fails to reach `finalized` is a `conflicting_state` **security event, not a
retry**.

For a claim write, rebuilding ends with either an identical idempotent anchor
or a scoped already-claimed refusal. Both are correct, and the event is still
reported. Faster finality is a roadmap item, not a correctness dependency.

## Errors and their retry semantics

Clients classify custom program errors **from the chain itself**, as in BNB.
Cardano instead classifies them off-chain. One class is new and is the most
common practical failure.

| Class | Condition | Client behavior |
|---|---|---|
| refresh-and-rebuild | Stale campaign revision; epoch not current | Re-resolve, re-consent to the revision, rebuild the presentation, and resubmit. A new epoch derives a new nullifier |
| retry-as-transport | **Blockhash expired**, or the transaction was dropped before inclusion | The transaction did not execute and consumed nothing, so resubmission is correct. A durable nonce is the profile option for slow provers on poor connections |
| terminal scoped refusal | Nullifier account already exists in this campaign and epoch | Show that the entitlement was claimed in this scope. Never reveal a person identifier or retry |
| refuse-as-defect | Invalid proof, out-of-field value, unknown policy, unauthorized signer | A generator or tooling bug; retry cannot help. Keep proof diagnostics private and out of logs |
| security event | Anchor contradicts one previously observed; confirmed anchor fails to reach `finalized` | Preserve evidence, raise the incident path, and never silently resubmit or overwrite |

Compute-unit exhaustion is deterministic, not transient. Retry fails until the
limit is raised, so this is refuse-as-defect. The profile fixes the requested
limit per instruction; clients do not guess.

## The operator manifest

The relayer's signed, byte-served manifest already exists for the
[Stellar notary](notary-stellar.md#the-live-operator-manifest). It would add the
charity profile, administered anchor deployments, and Solana network entries.
Those entries bind the Ed25519 operator identity to submission fee-payer
accounts and the admin account accepted by operator instructions.

Solana accounts also use Ed25519, removing the EVM entries' cross-key-type
awkwardness. The capabilities remain distinct. The manifest identity and
on-chain fee payer must not be conflated merely because they share a signature
scheme.

The normative client check gains one row. Client verification MUST compare the
declared admin with campaign state, the
declared program-data hash with the chain, and **the upgrade authority with
`None`**. Where SAS gates root updates, it MUST compare the declared credential
with the account the program reads. Otherwise declared powers cannot be matched
to program-enforced reality. Clients treat a deployment absent from the
manifest as nonexistent, regardless of chain state.

## Honest status

- **Nothing runs or specifies this binding.** There is no
  `UI-Charity-Solana.md`. `onym-system` contains only `Charity.md`,
  `UI-Charity.md`, and `UI-Charity-BNB.md`. No charity programs or circuits
  exist on any curve. The relayer has no Solana endpoints. Neither
  [`onym-contracts`](https://github.com/onymchat/onym-contracts) nor the relayer
  repository contains `solana` or `charity` in any source file. *(Repository
  state verified 21 August 2026.)*
- **Nothing is reused internally.** Unlike the Cardano plan, this starts without
  a working prover. BN254 Groth16, the circuits, programs, trusted setup, and
  Solana relayer backend are new. The curve is shared with the
  [BNB plan](charity-bnb.md), but the proof system and backend are not.
- **External components are candidate dependencies.** SAS supplies credentials,
  `groth16-solana` the verifier, Kora fee relaying, and ZK Compression the
  nullifier set if measurement supports it. The profile must name each external
  party, version, and failure mode.
- **Roadmap items are inputs, not commitments.** BN254 G2 and BLS12-381 syscalls
  could collapse the three curve stacks towards one. Larger
  transactions and lower rent could reduce per-claim state costs. This design
  requires none of them. The profile must re-verify activation status when
  written.
- `Charity.md` and `UI-Charity.md` are merged drafts, version 0.1 from August
  2026. The upstream `Charity.md` §6.8 question flagged by BNB about
  campaign-scoped fields in public claim anchors remains unresolved and binds
  this profile. This page adds another unresolved §6.8 question:
  [what settlement may publish](#settlement-is-where-this-binding-gets-hard).
- The design input is an August 2026 Solana implementation funding application.
  Like the July 2026 memorandum rejected as normative by `UI-Charity.md` §8.3,
  it is not a protocol dependency or evidence of funding, adoption, or
  implementation.

## Build order

Solana has its own dependency chain. It neither gates nor is gated by a sibling.

1. **Measure before specifying.** Two cheap results can invalidate the proof
   section. Benchmark the compiled eligibility circuit at its actual
   public-input layout. Compare it with the corresponding upstream figure under
   the same methodology and name the required verifier path. Prove byte-for-byte
   equivalence between in-circuit Poseidon and `sol_poseidon`. Publish the
   PDA-versus-compression comparison because it fixes the program account model.
2. **Profile, in two halves** as `charity/UI-Charity-Solana.md`. While step 1
   runs, specify account layouts and public fields, the two authorization
   gates, SAS bindings, deployment identity, upgrade authority, error taxonomy,
   and negative-PII fixtures across all five governed surfaces. Specify public
   inputs, proof encoding, and requested compute limits after step 1.
   `UI-Charity.md` §8.3 still applies: until the profile exists and passes
   conformance, “uses Solana” does not prove the boundary is satisfied.
3. **Circuits and ceremony.** Build BN254 eligibility constraints, including
   in-circuit expiry. Run a per-circuit Groth16 setup with a published
   transcript, identified participants, and named owner. An unauditable
   verifying-key origin is a forgery surface for real-aid claims.
4. **Prover.** Add the BN254 Groth16 backend to the mobile Rust FFI. It is new
   work shared with nothing and must preserve step 1's Poseidon parameters.
5. **Programs.** Build the anchor program and verifier. Publish a verifiable
   build and deploy with upgrade authority revoked.
6. **Audit.** Independently review circuits, verifier, and authority model for
   the shipped verifier path. Upstream coverage does not transfer.
7. **Relayer Solana backend.** Add transaction construction, fee-payer key
   management, blockhash and durable-nonce handling, commitment reconciliation,
   and fork watching. No other binding shares this work today.
8. **Declare, list, prove.** Add manifest entries, then discovery listing, then
   pass the fixture suite before binding any real campaign.
9. **Devnet pilot, then a mainnet-beta decision.** Run real campaigns end to end
   on devnet with green conformance vectors. Decide on mainnet only after the
   audit, matching the [Stellar](charity-stellar.md) and
   [Cardano](charity-cardano.md) final phases. Beneficiaries must not become the
   test surface for an unaudited verifier.

Settlement is not a numbered step. It is resolved during step 2 as a scope
decision: notary-only, or notary plus a separate settlement profile. The second
choice requires its own build chain, audit, and named legal counterparty.

## Open profile questions

The profile must resolve these questions:

1. **Settlement scope:** notary-only like all three siblings, or a separate
   on-chain settlement profile. If on-chain, which of the three publication
   shapes applies, and which custodial
   legal counterparty implements `refund-pending`, `refunded`, and `reversed`
   without a reversal primitive?
2. **Nullifier set shape:** does measured compute, state, and concurrency justify
   replacing the pilot PDA registry with ZK Compression and adding indexer and
   proof-service dependencies to the aid path?
3. **Verifier path and budget:** does the compiled circuit require plain
   Groth16 or BSB22, and what fixed requested compute limit follows?
4. **Trusted-setup governance:** who owns and runs each ceremony, who
   participates, where is its transcript published, and what happens to
   deployed campaigns when the circuit changes?
5. **Root-update authority:** SAS lookup at call time or a pinned signer set,
   and how can a client verify mutable authority without an unaffordable second
   network round-trip?
6. **Fee-payer concentration:** how many relayer keys exist, whether they are
   declared, and whether cross-campaign reuse expands the
   [gas-paying-submitter risk](charity-adversary.md#5-the-single-gas-paying-submitter).
   The count controls both privacy and throughput.
7. **Anchor batching:** as on BNB, how should audit granularity trade against
   timing correlation? On-chain settlement makes the trade apply to both money
   and claims.
8. **Statement separation from BNB:** because the curve is shared, separation
   depends on statement tag and profile ID. Different proof systems make
   cross-acceptance unlikely, but cross-rejection fixtures must prove it. They
   cannot be inherited from a cross-curve pair.

## Next steps

- [Charity](charity.md) — the abstract seat this would bind.
- [BNB Chain](charity-bnb.md) — the merged specification whose section shape
  this page mirrors, sharing its curve but not its proof system.
- [Cardano](charity-cardano.md) — the sibling with no sender and a prover this
  binding cannot reuse.
- [Stellar/Soroban](charity-stellar.md) — the planned reference binding whose
  relayer and manifest discipline this plan reuses.
- [The adversary's view](charity-adversary.md) — the public trail, with Solana's
  log, fee-payer, and settlement deltas.
- [Who holds which role](charity-roles.md) — abstract roles mapped to concrete
  parties, including unassigned roles.
