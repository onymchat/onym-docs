# Notary — Stellar/Soroban

The reference notary implementation, and the only one with running
code: every Onym group that exists today lands here. TurboPLONK proofs
over BLS12-381, verified on-chain by Soroban host functions, submitted
through a relayer that pays the fees so your wallet doesn't have to
exist.

**Status:** Running alpha implementation on Stellar testnet; not
production-audited.

**Profile:** [`notary/UI-Notary-Stellar.md`](https://github.com/onymchat/onym-system/blob/main/notary/UI-Notary-Stellar.md)
(implements the [abstract notary contract](notary.md))
**Code:** [`onym-contracts`](https://github.com/onymchat/onym-contracts) (on-chain)
· [`onym-relayer`](https://github.com/onymchat/onym-relayer) (submission)
**Live:** Stellar testnet · `relayer.onym.app`

## The contracts

Each of the [five flavors](notary.md#five-flavors-of-group-state) is
implemented by the matching `plonk/sep-<flavor>` contract crate in
`onym-contracts`; the flavor names are the wire's `contractType`
values. Five WASMs are deployed to Stellar testnet.

Every deployed contract also has a deployment-time operator admin whose
only power is a switch gating new group *creation*. It cannot touch any
existing group. Member caps come from circuit depth baked into the
verifying keys, not from an on-chain counter anyone could edit.

## The live operator manifest

The relayer's operator declares its powers in a signed manifest served
byte-for-byte — the pattern the
[moderation authority](moderation.md) already uses for its terms.
This is live: `https://relayer.onym.app/manifest.json` serves the
signed manifest, with the detached signature at
`GET /manifest.json.sig`, verified at boot from wherever
`RELAYER_OPERATOR_MANIFEST` points
([#13](https://github.com/onymchat/onym-relayer/pull/13)). Signing
happens in CI
([`sign-manifest.yml`](https://github.com/onymchat/onym-relayer/blob/main/.github/workflows/sign-manifest.yml)):
the workflow signs the manifest source with the operator seed held in
Actions secrets, commits the exact signed bytes to `main` under
`manifest-signed/` so they are auditable in-repo, and then checks that
the live endpoint serves **exactly** those bytes — byte-paranoid on
purpose, because group bindings and entitlements pin the SHA-256 of
the served manifest. A `verify-operator-manifest` subcommand runs the
same check on any manifest/signature pair, and scheduled scripts watch
for drift, expiry, and submitter funding. When `validUntil` passes
(currently 2027-08-14), the service refuses to serve the stale bytes:
both routes degrade to 404 until a re-signed manifest ships.

The manifest declares exactly one implementation profile
(`stellar-soroban-sep-plonk-v1`) and one network (Stellar testnet,
fee-payer role) — the complete truth about this operator today. When
the [BNB backend](notary-bnb.md) is built, its `eip155` networks join
the same manifest; the manifest, not the app, is what says which
chains an operator serves. The live signed catalog at
`discovery.onym.app` already indexes this manifest — see
[Discovery — Static Snapshot / Ed25519](discovery-static-ed25519.md)
for the operator key fingerprint to compare on your TOFU screen.

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

## Honest limits

The profile documents are candid about the distance between spec and
code; the gaps most worth knowing:

- **Testnet only.** The five contracts are deployed to Stellar
  testnet; no mainnet deployment exists.
- **Stellar write receipts are thin.** The relayer's success response
  currently omits the transaction hash, so clients cannot yet run fully
  independent transaction reconciliation from it. (The
  [BNB plan](notary-bnb.md) makes the hash mandatory from its first
  release, plus an outcome-query endpoint; the Stellar path has
  neither yet.)
- **Payment is not wired.** The `PaymentRequired` / entitlement flow is
  specified but unimplemented; today the relayer just pays and may
  require a bearer token.
- **Reads mostly trust the relayer.** Independent read-provider
  selection and long-term evidence retention are specified, not built.
- **A post-quantum circuit family** (`pq/`, Plonky3 + FRI) is in
  progress and blocked on FRI host functions.

## Next steps

- [Notary](notary.md) — the technology-free contract this
  implementation answers to.
- [BNB Chain plan](notary-bnb.md) — the second implementation profile
  and what it will reuse from this one.
- [Deployment](../deployment.md) — how the reference deployment brings
  the relayer up.
