# Identity

A user's identity is a root secret only they hold, plus a vault that
derives from it every purpose-specific key their activity in Onym
needs — and nothing else ever sees the root. The vault holds custody,
answers capability requests, and owns consent, rotation, and recovery.
It is not an account, and no server issues it.

**Contract:** [`identity/UI-Identity.md`](https://github.com/onymchat/onym-system/blob/main/identity/UI-Identity.md)
**Implementations:** [BIP-39](identity-bip39.md) — the only profile
that exists today, and the only one with running code.

This page stays deliberately free of any one key-production method —
so does the contract. It "does not require a particular mnemonic
standard, derivation function, curve set, storage hardware, custody
arrangement, or recovery ceremony." A concrete implementation may use
a locally generated mnemonic, a hardware wallet, an operating-system
keystore, a threshold custody service, or another mechanism entirely,
as long as it satisfies the same custody, derivation, consent,
disclosure, and rotation semantics. Two profiles that both happen to
derive keys from a seed are not compatible just because of that —
compatibility means a requester gets the same capability semantics,
result formats, and disclosure guarantees from either one.

## The roles, kept apart on purpose

| Role | What it controls — and only that |
|---|---|
| **Identity holder** | Owns the identity: the root secret and its recovery material. |
| **Vault implementation author** | Publishes software that generates, stores, derives, and uses key material. Does not own identities created with it. |
| **Custody or hardware provider** | Owns the device or service that executes vault operations. Never becomes the identity. |
| **Capability requester** | The chat flow, a seat adapter, a billing coordinator, an external application — receives typed results, never key material. |
| **Association registry** | Owns namespace and issuance policy, asserts names or credentials. Doesn't control the identity it names. |
| **UI publisher** | Presents identities, consent, and routes. "A window, not an account authority." |

## What the vault must do

- Generate or import a root secret and derive purpose-specific keys
  under declared, versioned **derivation domains**.
- Authenticate the holder locally before use.
- Publish and update a signed public identity descriptor.
- Evaluate capability requests against local policy and explicit
  consent.
- Execute narrow operations — sign, decrypt, prove, derive — and
  return typed results, never key material.
- Maintain per-context and per-seat keys unlinkable by default.
- Record grants and support revocation.
- Perform rotation, recovery, and compromise procedures its
  implementation profile defines.

The contract names capabilities by what they let a requester *do*,
never by naming a key: `sign-transaction`, `prove-address-control`,
`decrypt-envelope`, `prove-credential`, `derive-context-address`,
`prove-membership`, `derive-seat-key`. `derive-seat-key` is worth
calling out specifically — it's a distinct, pseudonymous,
component-scoped, unlinkable access key, so paying for one seat's
service can never be correlated with another.

## The common identity surface

Every implementation profile exposes the same operation names,
whatever it does underneath them:

| Operation | What it does |
|---|---|
| `create` / `import` | Establish a root secret, freshly or from existing material. |
| `unlock` | Authenticate the holder locally before any operation runs. |
| `descriptor` | The signed public identity descriptor — verification methods, not keys. |
| `request` / `grant` / `revokeGrant` / `listGrants` | The capability port: a requester asks, the holder consents (or not), and grants can be listed and later revoked. |
| `rotate` | Move a compromised or aging key forward without abandoning the identity. |
| `revoke` | Declare a key or the whole identity no longer trusted. |
| `export` / `wipe` | Leave one implementation for another, or destroy the vault entirely. |

## Rotation, recovery, and compromise are questions, not answers

The abstract contract doesn't answer these — it requires every
implementation profile to: can operational keys rotate independently
of the root ("operational separation")? How far does a rotation reach —
does it touch every derived domain, or only one? What does a recovery
ceremony require of the holder? How is compromise *declared*, and to
whom? Is the worst case partial or catastrophic? A profile that leaves
any of these unanswered hasn't finished the contract, even if its key
derivation works perfectly.

## Security and privacy invariants

Two carry the whole design:

- **The root never crosses the boundary.** No capability result, no
  descriptor, no log line ever contains the root secret or lets one be
  reconstructed from what a requester received.
- **The vault is replaceable.** A holder can leave one implementation
  for another without losing the identity, because the contract
  defines the portable interface independently of any one vault's
  internals — export exists precisely so this is true in practice, not
  just in principle.

## What this seat admits it can't promise yet

- **Only one implementation profile exists.** [BIP-39](identity-bip39.md)
  is the current implementation; the contract explicitly leaves room
  for others — hardware-rooted keys, threshold custody, post-quantum
  algorithms — while preserving the same capability port. None of
  those exist today.
- **Compromise, in the one profile that ships, is total, not
  partial.** See [BIP-39's honest status](identity-bip39.md#honest-status)
  for what that costs in practice.
- **Unknown implementations are never inferred from a mnemonic word
  count or address format.** A requester either recognizes a
  declared, versioned profile, or it doesn't proceed — there is no
  best-effort guessing path.

## Next steps

- [Identity — BIP-39](identity-bip39.md) — the one concrete profile
  that exists, and where its recovery-phrase derivation actually lives
  in the client repos.
- [Backup](backup.md) — a neighboring seat that, in its one profile,
  currently shares this seat's root secret rather than generating its
  own; a backup restores history, never identity.
- Recovery trustee — owns the authority to act as a person when the
  root secret is lost. Named in the contract as a separate seat; no
  seat page exists here yet.
