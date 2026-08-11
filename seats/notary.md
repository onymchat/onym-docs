# Notary

Validates group shared-state transitions against zero-knowledge proofs
without storing conversations or a readable member list.

**Contract:** [`notary/UI-Notary.md`](https://github.com/onymchat/onym-system/blob/main/notary/UI-Notary.md)
· profile: [Stellar/Soroban](https://github.com/onymchat/onym-system/blob/main/notary/UI-Notary-Stellar.md)
**Code:** [`onym-contracts`](https://github.com/onymchat/onym-contracts) (on-chain)
· [`onym-relayer`](https://github.com/onymchat/onym-relayer) (submission)

## Five flavors of group state

Pick by who can advance it. Flavor is a deployment-time choice — state
does not migrate across flavors.

| Contract | Members | Admins | Who advances state | Proof |
|---|---|---|---|---|
| `sep-anarchy` | ≤ 2¹¹ | — | any member | 1 membership π |
| `sep-oneonone` | 2 | — | nobody, immutable | — |
| `sep-democracy` | ≤ 2¹¹ | — | K-of-N member quorum | K πs batched in 1 |
| `sep-oligarchy` | ≤ 2¹¹ | ≤ 32 | K-of-N admin quorum, admin tree hidden post-create | K πs batched in 1 |
| `sep-tyranny` | ≤ 2¹¹ | 1 | single pinned admin, cross-group unlinkable | single admin π |

Every contract also stores a deployment-time **operator** admin whose only
power is `set_restricted_mode` — gating group *creation*. It can never
advance or modify existing group state. Caps come from circuit depth in
the baked VKs, not an on-chain counter.

Today's flavor is `plonk/` (TurboPlonk + BLS12-381, EF KZG 2023 SRS,
n=32768). `pq/` (Plonky3 + FRI, transparent setup) is in progress and
blocked on FRI host functions.

## Relayer

Fee-paying HTTP front door. `POST /` with:

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

`network` accepts `testnet`, `public`, `mainnet`. `contractID` must be
allowlisted for that `contractType` on that network. Byte fields may be
base64 or hex; `BytesN` arguments are forwarded as hex.

Allowed functions: `create_group`, `create_oligarchy_group`,
`update_commitment`, `verify_membership`, `get_commitment`, `get_history`,
`bump_group_ttl`, and tyranny-only `get_admin_commitment`.
`set_restricted_mode` is exposed only when `RELAYER_AUTH_TOKENS` is set,
because the relayer signs that admin operation.

The allowlist is not baked into the image. The relayer pulls
`contracts-manifest.json` from the `onym-contracts` latest release on boot
and on a 15-minute timer — the manifest is cumulative across all historical
releases. `POST /admin/refresh` (bearer auth) forces it; the contracts
release workflow calls that at the tail of every release.

## Build and test

```sh
cd plonk/sep-anarchy
cargo build --release --target wasm32v1-none
cargo test --lib
```

Toolchains are pinned per crate (`rust-toolchain.toml`): plonk contracts
1.91.0 / `soroban-sdk 26.0.0-rc.1`, pq 1.95.0 / `soroban-sdk 26.0.0`,
prover and `sep-*-ffi` 1.88.0.

Relayer: `cp .env.example .env`, set `RELAYER_SECRET_KEY`, then `./run.sh`.

## Fixtures

`plonk/verifier/tests/fixtures/` holds baked VK, canonical proof and
public-input bytes. CI re-bakes and byte-compares on every PR, so prover
drift fails the build. Regenerate deliberately:

```sh
cd plonk/prover && STELLAR_REGEN_FIXTURES=1 cargo test --release --lib \
  plonk_verifier_fixtures_match_or_regenerate
```

The mobile SDKs verify against the same SHA pins, so a divergence surfaces
across all consumers.

## Release

`gh workflow run release.yml -f tag=vX.Y.Z` builds five WASMs with
`stellar contract build --optimize`, stamps `--meta source_repo` and
`--meta home_domain='onym.chat'`, deploys to testnet, captures per-op fees
into the release body, and republishes `contracts-manifest.json`.
