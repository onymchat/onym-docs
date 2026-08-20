# Identity — BIP-39

The only implementation profile that exists today, and the root every
running client actually uses: one twelve-word mnemonic, generated
on-device, deterministically fans out into every purpose-specific key
the identity contract requires.

**Profile:** [`identity/UI-Identity-BIP39.md`](https://github.com/onymchat/onym-system/blob/main/identity/UI-Identity-BIP39.md)
(`onym:identity-implementation:bip39-multikey-v1`, mapping the abstract
`onym:identity-profile:vault-capability-v1`) — implements the abstract
[Identity](identity.md) contract.
**Code:** [`onym-ios`](https://github.com/onymchat/onym-ios) —
`Packages/OnymFoundation` (derivation), `Packages/OnymIdentity`
(`IdentityRepository`)
· [`onym-android`](https://github.com/onymchat/onym-android) —
`modules/foundation` (derivation, ported 1:1 from iOS), `modules/identity`
(`IdentityRepository`)
· [`onym-sdk-swift`](https://github.com/onymchat/onym-sdk-swift) /
[`onym-sdk-kotlin`](https://github.com/onymchat/onym-sdk-kotlin) — the
shared cryptographic primitives the derived keys are used through.

There is no identity server. This profile is a device-side library
plus each client's `IdentityRepository` and recovery-phrase flow.

## One seed, five domains

A twelve-word mnemonic (128 bits of on-device CSPRNG entropy) becomes a
seed via standard BIP-39. From that one seed, domain-separated
derivation produces one key per declared purpose — not a single
general-purpose keypair reused everywhere:

| Domain | Algorithm | Purpose |
|---|---|---|
| Event operations | secp256k1, Schnorr (BIP-340) | Outer, Nostr-compatible event signing |
| Membership proofs | BLS12-381 | Private group-membership trees, ZK proof witnesses |
| Ledger authorization | Ed25519 | Stellar-facing authorization; the public key doubles as the Stellar account ID |
| Sealed delivery | X25519 | Sealed inbox envelopes and invitation delivery |
| Transport discovery | Derived tag | Inbox discovery on couriers, without a readable account name |

This is the concrete answer to the abstract contract's "required set
of keys" — five domains today, chosen because five different
downstream consumers (event signing, group-membership proofs, a
Stellar-facing ledger, sealed delivery, courier discovery) each need a
key with different algebraic properties. A sixth domain for a chain
this profile doesn't yet address would need its own versioned adapter,
not a repurposed existing key — the profile is explicit that it
provides no general chain fan-out.

## Where the derivation actually lives

Both clients implement BIP-39 natively, not through a shared crate:

- iOS: `Packages/OnymFoundation/Sources/OnymFoundation/Bip39.swift` —
  mnemonic generation, validation, and seed derivation (PBKDF2-HMAC-SHA512
  over `"mnemonic"` + passphrase salt, 2048 iterations), then
  HKDF-SHA256 per domain with named info strings (`nostr-secp256k1-v1`,
  `bls12-381-v1`, and siblings for the other three).
- Android: `modules/foundation/src/main/kotlin/app/onym/android/foundation/Bip39.kt`,
  documented in its own header as ported 1:1 from the Swift file — with
  a warning that any change here must be matched on iOS or recovery
  phrases stop working across clients.
- `IdentityRepository` (iOS: `Packages/OnymIdentity`; Android:
  `modules/identity`) is the orchestrator that runs the full chain —
  mnemonic → seed → five domain keys — and the persistent reactive
  store the rest of each client reads from.

`onym-sdk-swift`/`onym-sdk-kotlin` wrap the same per-type PLONK FFI
staticlibs built from `onym-contracts/plonk/sep-*-ffi`, so proofs and
hashes stay byte-identical across platforms; the derived keys from
`IdentityRepository` are what get passed through these primitives —
`Common.nostrDerivePublicKey` / `nostrSignEventId` for the event-ops
domain, `Anarchy`/`OneOnOne`/`Tyranny`'s `prove*` functions for the
membership-proof domain's BLS witnesses. The SDKs themselves hold no
mnemonic or seed logic; they only operate on keys the vault already
derived.

## Conformance, as this profile currently defines it

| Contract concept | This profile's answer |
|---|---|
| Root secret | A 12-word BIP-39 mnemonic |
| Storage | Hardware-backed platform keystore, gated on OS authentication |
| Recovery | Full re-derivation — re-entering the mnemonic on a new device reproduces every domain key |
| Export | Knowing the mnemonic |

## Honest status

The profile's own spec names eleven gaps against the abstract
contract; the ones worth knowing before building against this profile:

- **The derivation tree isn't frozen yet.** It's implemented and the
  two clients are kept in lockstep by hand, but there's no published,
  versioned spec with cross-platform test vectors a third
  implementation could conform to. This is named as the single
  highest-priority gap.
- **There is no capability port.** No typed requests, no requester
  identities, no grants, no consent prompts, no canonical intent
  binding — the `request`/`grant`/`revokeGrant`/`listGrants` surface
  the abstract contract describes doesn't exist in this profile yet.
- **No `IdentityDescriptor` is published.** No verification methods,
  no rotation signals, no rollback detection.
- **Rotation and revocation are absent.** Every domain key is fixed by
  the mnemonic; there is no operational/recovery key split. The
  contract's abstract §13 invariant "the vault is replaceable" holds
  at the seed level (re-derive elsewhere) but not at the per-key
  level.
- **Compromise is total, not partial.** Holding the mnemonic means
  holding every domain simultaneously — messages, group-membership
  witnesses, and ledger authorization all at once. The spec calls this
  "currently a property, not a bug, of the profile," not a design
  goal.
- **Seat keys aren't derived**, which blocks conforming paid-seat
  authentication (`derive-seat-key` from the abstract contract).
- **No per-context addresses within a domain** — activity within one
  domain is linkable by default, unlike the unlinkable-by-default
  requirement the abstract contract states for per-context and
  per-seat keys.
- **`prove-address-control` and `prove-credential` have no standalone
  formats** yet, and chain adapters beyond the Stellar-facing Ed25519
  domain are unspecified.
- **Conformance today rests on the two clients mirroring each other**,
  not on a fixture suite a third implementation could pass —
  cross-platform test vectors (mnemonic→seed, seed→per-domain key,
  inbox-tag, per-domain signing operations, BLS witnesses, Stellar
  account-ID encoding, multi-identity isolation, restore round-trip,
  negative vectors) are specified but not published.

## Next steps

- [Identity](identity.md) — the technology-free contract this profile
  implements.
- [Backup — Object-HTTP](backup-object-http.md) — the one other
  profile in this book that derives its own keys from this same BIP-39
  seed, and states plainly what sharing a root costs.
- Recovery trustee — the seat that would own recovery when the
  mnemonic itself is lost, not merely re-entered. No seat page exists
  here yet.
