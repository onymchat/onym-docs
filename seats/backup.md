# Backup

*Seat page, draft 0.1 — 19 August 2026.*

A phone gets lost, stolen, or replaced, and with it goes a person's
whole history — unless something durable is sitting outside the device.
That's what this seat is for: a copy of a device's local state, held by
an operator who never gets to read it. The operator counts bytes and
enforces a deadline. It cannot open what it stores, cannot hand it back
to anyone but the person who sealed it, and cannot quietly reset access
when that person loses their key — because any mechanism that let it do
so would also let it read the archive.

**Contract:** [`backup/UI-Backup.md`](https://github.com/onymchat/onym-system/blob/main/backup/UI-Backup.md)
— the technology-free boundary this page summarizes.
**Implementation:** [`backup/UI-Backup-Object-HTTP.md`](https://github.com/onymchat/onym-system/blob/main/backup/UI-Backup-Object-HTTP.md)
— object storage over HTTPS, the first profile and so far the only one.
Both are drafts, and **no backup code exists in any Onym repository**:
no conforming adapter, no conforming operator, no fixture suite.

This boundary is deliberately narrow, and two neighbors mark its edges.
Identity and recovery own the root secret and the authority to act as a
person — a backup restores *history*, never *identity*, and a snapshot
can never legally contain seed material, a recovery artifact, or a
trustee share. Live attachments belong to blob storage and messages in
flight to the courier seat; this boundary is a device's own archive of
what it has already received.

## The roles, kept apart on purpose

| Role | What it controls — and only that |
|---|---|
| **UI owner** | What's eligible for backup, the consent surface, scheduling, local preparation, billing, and which profiles are supported. |
| **Application-protocol author** | Snapshot composition, sealing, key derivation, and restore validation. |
| **Backup-adapter author** | Maps the common backup port to one concrete retention protocol. |
| **Backup operator** | Endpoints, capacity, declared retention, erasure execution, jurisdictions, sub-processors, and its own payment model. |
| **User** | Whether a backup exists at all, under which declared terms, for how long, and when it's destroyed. |

One organization can hold several of these, but their authorities stay
separate: a person can switch UI and still restore an old snapshot, a UI
can use a third-party operator without importing its SDK, and an
operator can serve several frontends without learning what any of them
store.

## What crosses the boundary — and what never does

Two interfaces make up the seat: a local **UI ↔ backup adapter** port
that carries sealed bytes, snapshot references, policy bindings, restore
authorizations, erasure requests, and typed outcomes; and a network
**adapter ↔ operator** implementation profile that defines framing,
authorization, payment refusal, receipts, and export.

Neither interface may ever carry plaintext state, message plaintext,
contact labels, filenames, group secrets, decryption keys, recovery
artifacts, or seed material. A restorable snapshot necessarily
*contains* group secrets and content keys — that's what makes it a
restore instead of a transcript — so the rule the contract states is
**opacity**, not literal absence: nothing sensitive appears in the
clear, in metadata, in a locator, in a receipt, or in a log.

The operator sees, and is meant to see, only: a holder handle, a sealed
byte count, a digest, and a date.

## How a snapshot moves

```text
UI process
  |  user consent + schedule
  v
application backup composer
  |-- select eligible local state
  |-- seal under fresh key material derived from holder-held input
  |-- compute the snapshot reference over the sealed bytes
  |-- bind the operator's declared-policy digest
  `-- hand opaque bytes to the backup adapter
            |
            v
      backup adapter — map, authorize, verify, normalize
            |
            v
      backup operator — retain, serve, refuse, erase within declared policy
```

A successful operator response is evidence only of the operator's own
declared action at that moment — never a promise about future
availability, complete erasure, or a copy held anywhere else.

## Choices worth noting

- **The seed is the root, not a device-scoped key.** A snapshot sealed
  under key material a device can't export is unrestorable on exactly
  the device that will ever need it — a replacement. A client whose local
  storage is device-bound has to decrypt and re-seal, not copy its
  database files.
- **Every snapshot draws a fresh salt.** Keying is never convergent or
  content-derived, so two people sealing identical archives produce
  unrelated ciphertext and unrelated digests — nobody can confirm a
  holder possesses a known file. The cost is real: deduplication across
  holders becomes permanently impossible, and storage is linear in
  holders. That cost belongs in the operator's pricing, not engineered
  away later.
- **Restore proves possession, never identity.** The access key lives
  with the person, derived separately from their identity signing key.
  The operator verifies a proof and serves bytes — it holds nothing that
  opens a snapshot, and a lost access key means a permanently unreadable
  archive. There is no operator recourse, because every mechanism that
  would provide one — an escrow, a wrapped key, a support-driven reset —
  is also a mechanism for reading the archive.
- **Preflight is mandatory, and it's cheap.** A payment refusal has to
  cost one small request, never a completed multi-hundred-megabyte
  upload.
- **Export never consults entitlements.** Not "consult and allow" —
  no access at all. It's the only way the promise that a lapsed payment
  never holds a person's own history hostage survives future edits to
  the code.
- **Padding follows a Padmé bucket, not a power-of-two ladder.** Overhead
  stays under about 12% instead of potentially doubling stored bytes,
  which matters once storage is already linear in holders and can't be
  deduplicated away.

## Terms bind forward, never backward

A `BackupTerms` document is signed, content-addressed, and pinned into
every snapshot accepted under it. An operator can publish new terms at
any time, but a retained snapshot keeps the terms it was accepted
under — no new term may extend its retention, narrow its erasure scope,
add a jurisdiction or sub-processor, or remove its export path. An
operator that can no longer honor a pinned promise has to offer export
and erasure, not a unilateral restatement.

Lapse follows the same discipline: notice, then a grace period during
which download, export, and erasure all keep working, then the declared
post-grace behavior. Silent deletion on a failed charge doesn't conform,
and neither does withholding export until arrears are paid.

## What this seat admits it can't promise

- **Retention is a declaration, not a proof.** `retained` reflects the
  operator's own statement at one moment; a snapshot can still be lost
  under best-effort service.
- **Erasure is narrow and honest.** A receipt covers the primary copy,
  the declared replicas, and the operator's own historical copies — it
  can't reach recipients, observers, or copies the person made
  themselves. `excludedScope` is mandatory on every receipt, never
  decorative.
- **A backup extends a cost other people never chose.** A snapshot of
  one device extends the lifetime of both sides of every conversation in
  it, under a jurisdiction the other participants never selected. The
  contract can't give them a veto without holding a person's own history
  hostage to everyone they ever spoke with — so it requires the opposite
  discipline instead: the choice is opt-in, the terms are visible, and
  the consent surface says in plain words what the snapshot does for
  everyone in it.

## Where the code is

Nowhere, yet. The abstract contract and its one implementation profile
are both drafts, and the profile says so directly: no conforming
adapter, no conforming operator, no fixtures exist on either side.

The object-HTTP profile itself names what it still leaves open:
incremental upload that stays verifiable without leaking a change map to
the operator; a re-bind path after access-key rotation, so rotating
doesn't strand every snapshot sealed under the old key; and a disclosure
pattern for the third-party-cost problem above that's honest without
being unusable. Until fixtures land, the fallback rule already in force
elsewhere in the system stands: local state stays out of any cloud
backup unless the person explicitly includes it.

## Next steps

- **Identity and recovery** own the root secret this seat is built
  around but never carries — no seat page exists yet for either.
- [Courier](courier.md) — carries messages and blobs in motion; this
  seat is the device's own archive of what it already received, not a
  substitute route for either.
