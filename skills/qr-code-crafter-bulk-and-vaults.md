---
name: Bulk generation and capability-managed vaults
description: >-
  Produce a ZIP batch of up to 50 static QR files with a manifest, and organize up to 50 dynamic
  redirects under a single vault capability with folders, tags and aggregate analytics.
api: openapi/qr-code-crafter-openapi-original.json
operations:
  - generateBulkQr
  - createDynamicQrVault
  - getDynamicQrVault
  - updateDynamicQrVault
  - deleteDynamicQrVault
  - createDynamicQrVaultChild
  - updateDynamicQrVaultChild
  - getDynamicQrVaultChildAnalytics
  - deleteDynamicQrVaultChild
generated: '2026-08-11'
method: generated
source: >-
  Grounded in openapi/qr-code-crafter-openapi-original.json (operationIds verified verbatim),
  data-model/qr-code-crafter-data-model.yml and errors/qr-code-crafter-problem-types.yml.
---

# Bulk generation and capability-managed vaults

Two different batch surfaces. Bulk generation is stateless and returns files. Vaults are stateful and
return a capability. Do not confuse them.

## Bulk static ZIP — `generateBulkQr`

`POST /.netlify/functions/generate-qr-bulk`.

- Up to **50 rows** per request, as CSV text or as row objects with `value` and optional `filename`.
- One shared format and style applies to the whole ZIP — you cannot mix formats in a batch.
- Send `Accept: application/zip` for raw ZIP bytes, or take JSON for base64 ZIP metadata.
- The archive contains `manifest.csv` and `manifest.json`; each row carries `filename`, `value`,
  `format`, `status`, `bytes` and a `message`.

**Read the manifest before publishing anything.** A batch can partially succeed — per-row `status`
is the truth, not the HTTP status. Rows are normalized and empty or oversized rows are rejected
before generation.

Batch cost is the sum of its rows against a 180-unit-per-minute budget, so 50 PDFs (6 units each =
300) cannot fit in one window and will return `429`. Prefer `svg` (1 unit) for large batches and
split heavier `pdf`, `jpg`, `webp` or large-size batches. `413` means the request or the generated ZIP
exceeded its size ceiling — split it.

## Vaults — one capability for many redirects

A vault is the answer to "I have 40 dynamic codes and 40 unrecoverable tokens." It holds up to 50
child records under **one** vault capability; child credentials are never issued or exposed.

### Create — `createDynamicQrVault`

`POST /api/dynamic-qr-vaults`. Returns the vault, `ETag`, `X-Dynamic-QR-Vault-Version` and the
**one-time** vault `managementToken`. Same rule as a standalone record: persist it immediately, it is
never shown again, and it authorizes the vault *and everything inside it*.

An empty vault carries `emptyExpiresAt` and is garbage-collected on its own — that is how the provider
bounds abandoned state without accounts. Put a child in it or expect it to disappear.

### Add and edit children

- `createDynamicQrVaultChild` — `POST /api/dynamic-qr-vaults/{vaultId}/qr`
- `updateDynamicQrVaultChild` — `PATCH /api/dynamic-qr-vaults/{vaultId}/qr/{slug}`
- `deleteDynamicQrVaultChild` — `DELETE /api/dynamic-qr-vaults/{vaultId}/qr/{slug}/analytics`

Every one of these requires the vault bearer token **and** `If-Match`. Children add `folder` and
`tags` on top of the standard record fields — the only organizational metadata anywhere in this API.

Mind the validators: the vault has its own version (`X-Dynamic-QR-Vault-Version`) and each child has
its own (`X-Dynamic-QR-Version`). They are distinct. Sending a vault validator on a child write
returns `412`.

### Read analytics — `getDynamicQrVaultChildAnalytics`

`GET /api/dynamic-qr-vaults/{vaultId}/qr/{slug}/analytics`.

Aggregate only: a lifetime `totalScans` and UTC `daily` buckets, `approximate: true`, retained 90 days,
capped at 100,000 counts per day. There are no unique visitors, IPs, referrers, user agents,
locations or devices — not withheld, never collected. If a stakeholder needs attribution, put UTM
parameters on the destination and read them in first-party analytics after the page loads.

Check `available` before reading counts; the whole property is omitted when analytics are disabled
for the deployment.

### Delete — `deleteDynamicQrVault`

`DELETE /api/dynamic-qr-vaults/{vaultId}` with token + `If-Match`. Returns `409` if the vault still
owns children. Delete every child first.

## Failure modes specific to this surface

| Status | Meaning |
|---|---|
| `404` | The child is not owned by this vault — check `{vaultId}`/`{slug}` pairing |
| `409` | Vault is not empty |
| `412` | Wrong or stale validator; note vault and child validators are separate |
| `413` | Bulk request or generated ZIP too large |
| `503` | Vault storage unavailable or at capacity |
