# Seat map

Onym separates powers that a conventional service keeps under one
operator. A **seat** is one authority boundary. The names inside it are
the independently ownable **roles** that participate in that boundary.
A company may occupy several roles, but doing so does not merge their
authority.

![Onym seat map: a person or group chooses seven seats, with the independently replaceable roles listed inside each seat.](assets/seat-map.svg)

The lines show choice or interaction, not ownership. The person or group
in the middle is deliberately **not a seat**: identity stays with its
holder, and no seat has authority merely because another seat connects
to it.

## Read each boundary

- **[Identity](seats/identity.md):** identity holder, vault
  implementation author, custody or hardware provider, capability
  requester, association registry, and UI publisher.
- **[Discovery](seats/discovery.md):** instance operator, Discovery
  provider, catalog sponsor, auditor or attestation issuer, client, and
  user or group.
- **[Courier](seats/courier.md):** UI owner, application-protocol author,
  adapter author, courier operator, and user or group.
- **[Notary](seats/notary.md):** UI owner, policy author, notary operator,
  submission provider, read provider, and group.
- **[Moderation](seats/moderation.md):** user, reporter, moderation
  authority, interface, device-mark platform, and accused.
- **[Backup](seats/backup.md):** UI owner, application-protocol author,
  backup-adapter author, backup operator, and user.
- **[Charity](seats/charity.md):** user application, charity operator,
  organization credential issuer, eligibility issuer, financial
  provider, notary, and auditor or report issuer.

Each seat page defines what every role controls—and, just as
importantly, what it does not control. Implementation pages then bind
that abstract division to concrete software, services, and operators.

## How to read the vocabulary

| Term | Meaning |
|---|---|
| **Seat** | A replaceable authority boundary with its own contract. |
| **Role** | One independently attributable responsibility inside that boundary. |
| **Holder** | The person or organization occupying a role in one concrete deployment. |
| **Implementation profile** | The technology-specific rules that realize the abstract seat. |
| **Instance** | One concrete deployment or offer, normally described by a signed manifest. |

The map describes the abstract contracts, not current deployment
status. See [What exists today](README.md#what-exists-today) for the
dated implementation map.
