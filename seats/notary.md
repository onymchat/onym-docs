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
· profiles: [Stellar/Soroban](https://github.com/onymchat/onym-system/blob/main/notary/UI-Notary-Stellar.md) (running today),
[BNB Chain](https://github.com/onymchat/onym-system/blob/main/notary/UI-Notary-BNB.md) (accepted design, unbuilt)
**Code:** [`onym-contracts`](https://github.com/onymchat/onym-contracts) (on-chain)
· [`onym-relayer`](https://github.com/onymchat/onym-relayer) (submission)

## The words you'll keep seeing

| Term | What it means |
|---|---|
| **Commitment** | A 32-byte opaque value that stands for the group's membership and secrets. The chain stores it; only members can open it. |
| **Epoch** | The group's revision counter. Every accepted change increments it by exactly one — no gaps, no rewinds. |
| **Flavor** | The governance rule chosen at creation: who is allowed to advance the group's state. Five exist (see below). |
| **Proof** | A zero-knowledge proof that the proposed new commitment follows from the old one under the flavor's rule — without revealing who acted. |
| **Circuit type** | Which proof system and chain family the proofs target. Today `plonk` on Stellar (BLS12-381); the accepted BN254 variant targets BNB Chain. |
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

These flavor names are the wire's `contractType` values; each is
implemented by the matching `plonk/sep-<flavor>` contract crate in
`onym-contracts`.

Every deployed contract also has a deployment-time operator admin whose
only power is a switch gating new group *creation*. It cannot touch any
existing group. Member caps come from circuit depth baked into the
verifying keys, not from an on-chain counter anyone could edit.

## How a state change flows

1. **Read first.** Your app fetches the group's current commitment and
   epoch from the chain and verifies what it read.
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

Today every group lands on **Stellar/Soroban** — TurboPLONK proofs
over BLS12-381, verified by Soroban host functions. It is the only
chain with running code, so for now there is no choice to make.

The accepted design adds one. A notary operator may run deployments on
**several blockchains at once**, and the contract says *you* will
choose — at group creation, alongside flavor — which one receives your
group's proofs. The first addition is **BNB Chain**: the same
governance rules re-proved over BN254 and verified by standard Solidity
verifiers. It is a published design; none of it is built yet.

The design requires the choice be shown with its real consequences —
which public ledger your group's (opaque) activity lands on, its fees,
its finality — not just a curve name. Once made, it is pinned into the
group binding like everything else. Groups do not migrate between
chains; until a governed migration protocol exists, a re-created group
is a *new* group.

Today [Discovery](discovery.md) lists Soroban relayers — a name, a URL,
and supported Stellar networks. In the accepted design it grows into a
catalog of notaries run by **different parties**, each declaring
exactly which chains it supports in a signed manifest. Some operators
will support only one — an honest, declared condition — and the app
will offer only combinations your chosen operator actually declares.
It must never paper over a gap by silently switching you to a
different operator.

## The operator is a clerk, not a king

The relayer's operator holds real but narrow powers. In the accepted
design it declares them in a signed manifest served byte-for-byte,
adopting the pattern the [moderation authority](moderation.md) already
uses for its terms; today the relayer serves no such manifest, and the
powers live only in the contract documents and the code. Either way,
the governing invariant:

> The operator's key can pay for, submit, and gate the creation of
> group state. It can never author a group transition.

Concretely, the operator may refuse to submit (you can use another
relayer), require payment, rate-limit, and flip the creation gate. It
cannot advance any group's commitment, forge or substitute proofs, move
a group's binding, or make an invalid operation valid by accepting
money for it. These limits are enforced by the contract, not promised
by policy.

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

## Honest limits

The contract documents are candid about the distance between spec and
code; the gaps most worth knowing:

- **The entire BNB path is unbuilt.** No Solidity contracts, no BN254
  circuits, no EVM support in the relayer. The profile is an accepted
  design whose gaps list is the work plan.
- **Stellar write receipts are thin.** The relayer's success response
  currently omits the transaction hash, so clients cannot yet run fully
  independent transaction reconciliation from it.
- **Payment is not wired.** The `PaymentRequired` / entitlement flow is
  specified but unimplemented; today the relayer just pays and may
  require a bearer token.
- **Reads mostly trust the relayer.** Independent read-provider
  selection and long-term evidence retention are specified, not built.
- **A post-quantum circuit family** (`pq/`, Plonky3 + FRI) is in
  progress and blocked on FRI host functions.

## For developers

The relayer speaks JSON over `POST /`:

```json
{
  "network": "testnet",
  "contractID": "C...",
  "contractType": "anarchy",
  "function": "update_commitment",
  "payload": {
    "group_id": "base64-or-hex-32-byte-group-id",
    "proof": "base64-or-hex-1601-byte-proof",
    "publicInputs": ["c-old", "epoch-old-be32", "c-new"]
  }
}
```

`network` accepts `testnet`, `public`, or `mainnet`. Byte fields may
arrive as base64 or hex; `BytesN` arguments are forwarded to the chain
as hex. `contractID` must be allowlisted for that `contractType` on
that `network`; the allowlist is the cumulative
`contracts-manifest.json` from the latest `onym-contracts` release,
pulled at boot and every 15 minutes (`POST /admin/refresh` with bearer
auth forces it). Allowed functions:
`create_group`, `create_oligarchy_group`, `update_commitment`,
`verify_membership`, `get_commitment`, `get_history`, `bump_group_ttl`,
tyranny-only `get_admin_commitment`, and — only when auth tokens are
configured — the operator's `set_restricted_mode`.

Build, fixtures, and release, in brief:

```sh
# contracts
cd plonk/sep-anarchy && cargo build --release --target wasm32v1-none && cargo test --lib

# relayer
cp .env.example .env   # set RELAYER_SECRET_KEY
./run.sh

# regenerate proof fixtures deliberately
cd plonk/prover && STELLAR_REGEN_FIXTURES=1 cargo test --release --lib \
  plonk_verifier_fixtures_match_or_regenerate
```

Toolchains are pinned per crate via `rust-toolchain.toml`: plonk
contracts 1.91.0 / `soroban-sdk 26.0.0-rc.1`, pq 1.95.0 /
`soroban-sdk 26.0.0`, prover and `sep-*-ffi` 1.88.0.

`plonk/verifier/tests/fixtures/` holds the baked verifying key and
canonical proof/public-input bytes; CI re-bakes and byte-compares them
on every PR, so prover drift fails the build. The mobile SDKs verify
against the same SHA pins, so a divergence surfaces across all
consumers.

Releases run `gh workflow run release.yml -f tag=vX.Y.Z`: five WASMs
built with `stellar contract build --optimize`, stamped with
`--meta source_repo` and `--meta home_domain='onym.chat'`, deployed to
testnet, per-op fees captured into the release body, and a republished
cumulative `contracts-manifest.json`.

## Next steps

- [Discovery](discovery.md) — how clients find relayers, contracts, and
  (eventually) competing notary operators.
- [Deployment](../deployment.md) — how the reference deployment brings
  the relayer up.
