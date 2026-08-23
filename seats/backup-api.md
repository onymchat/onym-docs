# Backup operator API

*Implementation contract derived from `onym-backup` commit `3bf81fc` —
23 August 2026.*

If you are implementing an adapter or generating a client, use the
[`onym-backup` OpenAPI document](../openapi/onym-backup.yaml). It describes
the HTTP surface that the reference operator actually serves.

## Which document wins?

The three documents answer different questions:

1. [`onym-backup`](https://github.com/onymchat/onym-backup) is the source of
   truth for the reference operator's routes, request and response bodies,
   headers, status codes, and error codes.
2. The [OpenAPI document](../openapi/onym-backup.yaml) is a reviewable,
   client-generation description derived from that implementation. Its
   version names the source commit. If it disagrees with the same commit of
   `onym-backup`, the implementation wins and the OpenAPI document is a bug.
3. `UI-Backup-Object-HTTP.md` explains the intended Object-HTTP design, while
   `UI-Backup.md` defines the technology-neutral seat boundary. Neither is a
   second wire schema for the deployed operator.

In particular, abstract outcomes such as `terms_regression` and
`erasure_unconfirmed` are conclusions an adapter or UI may derive. They are
not error codes returned by `onym-backup`.

## Current HTTP surface

Public documents:

- `GET /health`
- `GET /manifest.json`
- `GET /manifest.json.sig`
- `GET /profile.json`
- `GET /terms/{64hex}.json`
- `GET /terms/{64hex}.json.sig`

Holder-authenticated operations:

- `POST /v1/entitlements`
- `POST /v1/preflight`
- `PUT /v1/uploads/{uploadId}/chunks/{index}`
- `POST /v1/uploads/{uploadId}/commit`
- `GET /v1/snapshots`
- `GET /v1/snapshots/{64hex}`
- `POST /v1/erasures`
- `GET /v1/exports`
- `GET /v1/exports/{64hex}`
- `GET /v1/exports/receipts/{receiptId}`
- `GET /v1/operations/{operationId}`

There is no holder challenge/proof session, single-shot snapshot upload,
`DELETE /v1/snapshots/{digest}`, or tar-building `POST /v1/export` endpoint.
Every authenticated request instead carries request-bound Ed25519 proof
headers.

## Resolved wire-shape conflicts

These are the implementation answers to the contradictions reported by an
early adopter:

| Question | `onym-backup` answer |
|---|---|
| `ExportManifest.receipts` | Array of paths such as `receipts/<receiptId>.json`, not receipt objects. Fetch each object through `GET /v1/exports/receipts/{receiptId}`. |
| Erasure request | `POST /v1/erasures` with JSON `{ "operationId": "...", "scope": "all" }`; `scope` may instead be a full `sha256:<64hex>` identifier. |
| Erasure response | A JSON array of signed receipts directly, one per pinned `termsId`; there is no `{ "receipts": [...] }` wrapper. |
| Manifest `offers` | Array of `{ "offerId": "...", "model": "subscription" }` objects. A `402` deliberately carries an array of bare offer IDs inside `paymentRequired.offers`. |
| Concurrent-upload limit | `maximumConcurrentUploads` is not a manifest or error field. Open grants count against quota and a `quota_exceeded` error reports `openGrants` and `openGrantBytes`. |
| `PreflightRequest` | Requires `operationId`, `snapshotReference`, and `acceptedTermsId`; it has no `version` field. `supersedes` is optional. |
| `UploadGrant` | Contains `uploadId`, `chunkBytes`, `chunkCount`, `expiresAt`, `acceptedTermsId`, and inclusive-range `missingChunks`; it does not echo `operationId`. |
| Snapshot list | Includes `retained`, `retention_expired`, and remembered `erased` records. Each item carries a nested `snapshotReference`. |
| Invalid reference | The operator has no `invalid_reference` code. Malformed request references use `bad_request`; missing holder-scoped resources use their typed `*_not_found` code. |
| Manifest and profile fields | Only the fields emitted by `/manifest.json` and `/profile.json` in `onym-backup` are part of this operator API. Abstract examples do not add fields to either document. |
| Erasure receipt identity | Every receipt carries `receiptId`, `operator`, `receiptVersion`, scope, covered snapshot references, deadlines, terms ID, and signature. |
| `terms_regression` | Client-side comparison result, not an operator response. The operator can return `terms_changed` when preflight pins terms that are no longer current. |
| `erasure_unconfirmed` | Client-side deadline interpretation, not an operator response. The operator returns a signed acknowledgement or a typed error. |

The OpenAPI file also pins details that are easy to get subtly wrong:
digest path parameters are bare lowercase hex even though JSON identifiers
include `sha256:`; proof timestamps are RFC3339; signed fields are
length-prefixed; and Ed25519 signatures use standard padded Base64.

## Keeping it aligned

Any change to routes or wire values starts in `onym-backup`. The accompanying
documentation change must then:

1. update `openapi/onym-backup.yaml` and its implementation commit version;
2. validate the OpenAPI document;
3. compare its method/path set with `operator/src/api.rs`; and
4. update this resolution page if a previously settled shape changes.

The shared conformance fixtures remain unwritten, so this precedence rule is
necessary but not a substitute for cross-client fixtures.
