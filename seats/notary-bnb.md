# Notary — BNB Chain

This page describes an implementation that **does not exist yet**.
The design is merged and settled — the profile document is the source
of truth, and its gaps list is the work plan — but there are no BN254
circuits, no Solidity contracts, and no EVM code in the relayer. Read
this as a map of what is coming and what it will change, not as
documentation of running code.

**Status:** Implementation plan with a merged specification; code not
implemented.

**Profile:** [`notary/UI-Notary-BNB.md`](https://github.com/onymchat/onym-system/blob/main/notary/UI-Notary-BNB.md)
(implements the [abstract notary contract](notary.md); sibling of the
[Stellar profile](notary-stellar.md), not its replacement)
**Code:** none.

## What it will be

The same notary seat on a second chain: the five governance flavors
re-proved as standard PLONK over **BN254** (KZG), verified by
toolchain-generated Solidity verifiers on **BNB Smart Chain** —
mainnet (chain ID 56) and testnet (97). opBNB is explicitly out of
scope: it has L2 finality semantics and would need its own profile.
The relayer gains an EVM backend beside its Stellar one, pays gas from
its own funded account (your device never holds an EVM key), and the
group's choice of chain is pinned into its binding at creation exactly
as the abstract contract requires.

Why BN254 rather than reusing the existing BLS12-381 circuits against
BSC's newer EIP-2537 precompiles: the BN254 pairing precompiles have
years of production exposure, verifier generation for them is automated
and audited in widely used toolchains, gas is cheaper, and a bespoke
hand-written verifier's audit surface is avoided. The cost is a real
circuit migration — new constraint systems, a new setup, new verifying
keys, and a second prover backend in the mobile Rust FFI. Because the
evidence scheme differs, profile IDs are not shared: a BLS12-381 proof
is not valid evidence under this profile, and a BN254 proof is not
valid under the Stellar one — fixtures must prove both rejections.

## Circuit type becomes a user choice

The contracts manifest gains an explicit `proofSystem` field
(`plonk-bn254-kzg` here; existing Stellar entries are retroactively
`turbo-plonk-bls12-381-kzg`, and an absent field means exactly that
value — a consumer must never default an *unknown* value to a known
one). At group creation the user selects the circuit type alongside
network and flavor, and the UI must:

- list only circuit types whose manifest entries it fully verified
  (profile, verifier anchors, code hashes, operator signature);
- show the choice with its practical consequences — which public
  ledger the group's (opaque) activity lands on, fees, finality,
  metadata exposure — not only a curve name;
- pin the selection into the group binding, sticky forever.

The selection is two-dimensional — which **operator**, and which of
that operator's **backends** — and the operator's signed manifest is
the complete truth about what it supports. A joiner never selects: the
invitation carries the pinned binding, and a client without a BN254
prover must refuse to join a `-bn254-` group rather than substitute
evidence.

## What changes versus Stellar

| | Stellar/Soroban (running) | BNB/EVM (planned) |
|---|---|---|
| Proof system | TurboPLONK over BLS12-381 | Standard PLONK over BN254 (KZG) |
| Verifier | Soroban host functions | Toolchain-generated Solidity contracts, deployed immutably |
| Deployment identity | `contractID` (`C...`) | EIP-155 chain ID + contract address + **runtime code hash** |
| Write receipt | Success response currently omits the transaction hash | Transaction hash **mandatory** from the first release, plus a `GET /operations/:operationId` outcome query (a gas-bumped replacement can supersede a hash; the operation ID is the stable key) |
| Errors | Stale state and malformed inputs collapse into one comparison | `StaleRevision` and `PublicInputsMismatch` are distinct custom errors — their retry guidance differs (refresh-and-rebuild vs refuse-as-defect) |
| State upkeep | Soroban TTL, `bump_group_ttl` | No rent: state persists; `maintainState` declared explicitly `false` |
| Finality | Ledger close | Receipt (`status == 1`) and the `finalized` checkpoint are distinct client-visible states; a receipt in a later-orphaned block is a security event, not a retry |
| Privacy | Opaque values on a less-indexed chain | Same opaque values, but calldata/state/events sit on public, heavily-indexed explorers, and the relayer's one submitting account links every group it serves — the selection UI must say so |

Two hardening rules worth calling out because they are easy to get
wrong: client-chosen 32-byte values (`group_id`, commitments) must be
generated **in-field** by rejection sampling and rejected by the
contract when out of range — modular reduction is non-injective and
would let evidence for one group authorize a second; and the verifier
address and operator-admin address are Solidity `immutable` values
baked into the runtime bytecode, so the binding's `runtimeCodeHash`
pin actually covers who verifies and who administers. Upgradeable
proxies are prohibited outright.

## The operator manifest already exists

The profile's second structural change — the relayer as a declared
notary-seat operator with a signed, byte-served manifest — is the one
piece that has already been built, on the Stellar side:
`relayer.onym.app/manifest.json` is
[live and CI-signed today](notary-stellar.md#the-live-operator-manifest).
What BNB adds to that manifest, not beside it:

- a second entry in `implementationProfiles`
  (`onym:notary-implementation:bnb-evm-sep-plonk-bn254-v1`);
- `eip155` entries in `networks`, each binding the Ed25519 operator
  identity to two distinct secp256k1 accounts: the `submitterAccount`
  that pays gas and the `adminAccount` whose `msg.sender` the
  contracts' `setRestrictedMode` accepts. Client deployment
  verification compares the manifest's `adminAccount` against the
  contract's `getOperatorAdmin()` — without that comparison, "declared
  powers match contract-enforced reality" is not verifiable;
- expiry and rotation semantics: `validUntil` ends *new* reliance
  only (existing bindings stay verifiable against the pinned bytes
  forever, and superseded versions stay retrievable by hash), and a
  key change is a rotation only when the old key signs an explicit
  rotation statement — an unsigned key change is a different operator.

A multi-chain operator lists every backend it runs; a single-chain
operator lists one, and that is the complete truth about it.
[Discovery](discovery.md) surfaces exactly what the manifest declares,
never more.

## The plan, in phases

Everything below is unbuilt; the order reflects the profile's
dependency chain:

1. **Circuits** — re-express the five flavors' predicates as BN254
   constraint systems with a new setup and verifying keys,
   cross-checked against the BLS12-381 circuits with shared logical
   fixtures, so the curve change provably does not change what is
   proven.
2. **Prover** — a BN254 PLONK backend in the mobile Rust FFI, beside
   the existing one.
3. **Contracts** — the Solidity `sep-*` family in `onym-contracts`:
   generated verifiers, custom errors, typed events, per-group proof
   replay records; written and audited.
4. **Relayer** — an EVM backend beside the Stellar CLI: secp256k1
   signing, nonce management and replacement (byte-identical calldata,
   only fee fields differ), gas estimation, receipt tracking,
   `chainId`/`proofSystem`-aware allowlisting, the mandatory
   transaction hash, and the operation outcome-query endpoint.
5. **Manifests** — `contracts-manifest.json` grows `chainId`,
   `proofSystem`, `verifier`, code-hash and profile fields (and gets
   signed); the operator manifest adds its BNB declarations as above.
6. **Clients** — EVM read path, receipt/event/finality reconciliation,
   the circuit-type selection UI with its honest consequences screen,
   and the missing BNB fields in the group binding.
7. **Discovery listing** — the operator's expanded manifest, indexed
   by the signed catalog, is what makes the new backend selectable.
8. **Conformance** — the end-to-end fixture chain (Swift/Kotlin input
   → Rust proof bytes → relayer JSON → ABI calldata → contract →
   RPC evidence → normalized snapshot). The profile is blunt that
   this is a **ship-blocker, not an afterthought**: prover toolchains
   disagree on byte encodings, so no client may ship until the
   fixtures pin the chosen toolchain's exact bytes.

Cross-chain migration is deliberately absent from the list: the
governed migration protocol is undefined, so until it exists a
re-created group is a new group linked by claim, not the same group.

## Honest status

- **Nothing on this page runs.** No BN254 circuits, setup, or
  verifying keys; no Solidity in `onym-contracts`; no EVM backend,
  allowlisting, or outcome endpoint in `onym-relayer`; no EVM read
  path or selection UI in either client; no BNB fixtures.
- **One §18 item has since landed**: the profile's gap list still says
  no operator manifest exists, but the manifest machinery was built
  and is live on the Stellar side — declaring, honestly, Stellar
  support only.
- The profile itself is a merged draft (0.1, August 2026): accepted
  decisions are settled, but byte-level details it deliberately leaves
  to the conformance suite (exact proof size and encoding) are not
  fixed anywhere yet.

## Next steps

- [Notary](notary.md) — the technology-free contract both
  implementations answer to.
- [Stellar/Soroban](notary-stellar.md) — the implementation that
  actually runs, and the operator-manifest pipeline BNB will extend.
- [Discovery](discovery.md) — where a second backend would become
  visible and selectable.
