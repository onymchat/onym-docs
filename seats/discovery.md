# Discovery

How does your app find a relayer to submit proofs through, a
moderation authority to consent to, a courier to speak over? Someone
has to publish that list — and whoever publishes it holds real power,
because a list decides who serves and who judges. The discovery seat
exists to make that power **signed, inspectable, and replaceable**:
catalogs you can verify, providers you can swap, and a guarantee that
absence from every catalog never blocks you from using an instance you
found yourself.

**Contract:** [`discovery/Discovery.md`](https://github.com/onymchat/onym-system/blob/main/discovery/Discovery.md)
**Implementations:** [Static snapshot / Ed25519](discovery-static-ed25519.md)
— the only profile that exists today, merged with running reference
code.

This contract "does not require a search engine, DNS system,
transport, storage layer, ranking algorithm, payment rail,
jurisdiction, or business model." A concrete implementation profile
must define canonical encodings, transport, pagination, signatures,
and test vectors — the current static-snapshot profile is one way to
answer those questions, not the only shape the contract allows.

An **instance** is one concrete operator, deployment, application,
institution, or offer for a seat's role, described by a signed
manifest. Discovery indexes *references* to those manifests — it never
becomes a root of trust, and catalog inclusion is a recommendation,
never protocol approval or proof of safety.

## The roles, kept apart on purpose

| Role | What it controls — and only that |
|---|---|
| **Instance operator** | One concrete seat instance, and its signed service or institutional manifest. |
| **Discovery provider** | Selects manifest references under its own published policy, signs catalog snapshots, answers queries, discloses ranking and commercial relationships. |
| **Catalog sponsor** | May fund a catalog for a declared audience. Funding doesn't silently change the provider's policy. |
| **Auditor / attestation issuer** | Signs scoped evidence about one exact instance or artifact. Discovery may cite it, but can't speak for its issuer. |
| **Client** | Verifies catalogs and manifests, applies local compatibility checks, preserves direct import, and shows the source and basis of every recommendation. |
| **User or group** | Chooses which Discovery sources to consult, and which downstream instance — if any — to select. |

One party may hold several roles, but the signed objects keep their
authorities separate. A Discovery provider that also operates, audits,
sponsors, or earns from a listed instance discloses that relationship
on the affected entry.

## How a listing actually reaches a user

1. An instance operator publishes a signed manifest under its
   destination seat's own contract.
2. A Discovery provider retrieves and verifies that manifest, applies
   its published inclusion policy, and records only a digest-bound
   reference — never a copy it could quietly edit.
3. The provider publishes a signed, expiring catalog snapshot.
4. The client obtains the provider's own manifest through direct
   import, a user-selected source, or a replaceable application
   default.
5. The client verifies provider identity, snapshot signature,
   sequence, policy digest, expiry, bounds, and requested seat type.
6. The client fetches candidate instance manifests, verifies their
   operator signatures and digests, and evaluates compatibility
   locally.
7. The UI shows catalog source, relevant evidence, commercial
   relationship, and material risk or compatibility information.
8. The user or group explicitly selects an instance under that
   instance's own seat contract.

Fetching, viewing, or ranking an entry grants the instance no
capability and creates no downstream order — discovery completion is
not service selection.

## What a catalog's policy has to disclose

Every catalog pins a human-readable, machine-identifiable policy
naming: eligible seat types, profiles, jurisdictions, and audiences;
required manifest freshness and availability checks; required
attestations and their issuers and maximum ages, if any; legal,
safety, quality, or accessibility criteria; listing, subscription,
sponsorship, referral, and common-ownership terms; ranking inputs and
their priority; removal, correction, appeal, and conflict processes;
review cadence and catalog expiry; and what the provider does *not*
verify.

A provider may curate — it isn't required to list every technically
compatible instance — but it can't claim completeness without a
reproducible source population and measurement time. Paid placement is
allowed only when disclosed and never described as an audit,
certification, or organic rank. A client that re-ranks locally must
not attribute the resulting order to the provider.

## No provider is a root authority

Every conforming client supports a direct manifest path — paste, scan,
file import, deep link, local configuration, or another profile's
mechanism — validated with the same signature, schema, and
compatibility checks a catalog result gets. A client may ship a
default Discovery provider, but only while it also lets the user add,
remove, and replace sources; never silently restores a removed source;
labels which source produced each recommendation; preserves direct
import; and never treats absence from the default as protocol
invalidity.

Discovery itself bootstraps through direct import or a replaceable
application default. A Discovery provider may list other Discovery
providers, but no provider is required to list itself, and no
recursive catalog establishes a root authority.

## What this seat admits it can't promise

- **Only one implementation profile exists.** The contract began as
  proposed architecture and names its implementation status as
  "tracked solely by that profile's §11" — there is no second profile
  to compare it against yet.
- **A catalog entry is a recommendation, not a guarantee.** Inclusion
  never certifies safety, availability, or that a listed operator is
  still the one it was when the catalog was signed.
- **Curation is allowed to be incomplete**, as long as a completeness
  claim, if made at all, defines a reproducible source population and
  measurement time rather than asserting coverage nobody checked.

## Next steps

- [Static snapshot / Ed25519](discovery-static-ed25519.md) — the one
  implementation profile that exists, what runs today versus what's
  still migrating onto it, and the operator fingerprints worth
  checking by hand.
- [Notary](notary.md) — the relayers a discovery catalog points at.
- [Moderation](moderation.md) — the authorities a discovery catalog
  feeds into a client's picker.
- [Deployment](../deployment.md) — how the reference deployment brings
  the server-side seats up.
