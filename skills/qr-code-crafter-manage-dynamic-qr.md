---
name: Create and manage a dynamic QR redirect
description: >-
  Create an editable HTTPS redirect on QR Code Crafter, hold its one-time capability token safely, and
  update, pause, resume, rotate or delete it using the mandatory If-Match version guard.
api: openapi/qr-code-crafter-openapi-original.json
operations:
  - createDynamicQr
  - getDynamicQr
  - updateDynamicQr
  - deleteDynamicQr
  - redirectDynamicQr
generated: '2026-08-11'
method: generated
source: >-
  Grounded in openapi/qr-code-crafter-openapi-original.json (operationIds verified verbatim),
  conventions/qr-code-crafter-conventions.yml and errors/qr-code-crafter-problem-types.yml.
---

# Create and manage a dynamic QR redirect

There are no accounts here. A 43-character capability token *is* the identity, the authorization and
the only way back to the record. Everything in this skill follows from that.

## 1. Create — `createDynamicQr`

`POST /api/dynamic-qr` with `{"destination": "https://...", "label": "optional"}`.

- `destination` must be **HTTPS** (`^https://`), max 2048 characters. HTTP is rejected with `400`.
- `label` is free text, max 80 characters. It is the only metadata a standalone record carries.

A `201` returns `record`, `redirectUrl`, `managementUrl` and `managementToken`, plus `ETag`,
`X-Dynamic-QR-Version` and `Location` headers.

**The `managementToken` is returned once and never again.** Persist it before you do anything else.
If you lose it the record is unreachable and unrecoverable — it will keep redirecting forever and you
cannot pause, edit or delete it.

**There is no idempotency key on create.** A retried POST makes a second, independent redirect. Treat
creation as at-most-once: if the call times out, do not blindly replay it — you have no way to
discover an orphaned record afterwards.

## 2. Read — `getDynamicQr`

`GET /api/dynamic-qr/{slug}` with `Authorization: Bearer {managementToken}`.

The `{slug}` is 22 characters and public — it appears in the printed `/r/{slug}` URL. It is **not** an
authorization secret; only the bearer token authorizes.

Read the response `ETag` and `X-Dynamic-QR-Version` here. You need one of them for every write.
`analytics` is present only when the deployment has aggregate counts enabled — the property is omitted
entirely when it is off, so check for presence rather than assuming a zero.

## 3. Write — `updateDynamicQr`

`PATCH /api/dynamic-qr/{slug}` with:

- `Authorization: Bearer {managementToken}`
- `If-Match: <the exact X-Dynamic-QR-Version or ETag value>` — **required**
- `X-Dynamic-QR-Version: <same value>` — send it too. The provider mirrors the validator onto this
  header specifically because some proxies rewrite `ETag`.
- `Content-Type: application/json`

Body must have at least one of `destination`, `label`, `status` (`active`|`paused`), `rotateToken`.

This is the retry-safe part of the API. A replayed PATCH carrying a stale validator returns `412` and
changes nothing, so you can retry a timed-out update without fear of applying it twice. Always
re-`GET` after a `412` and replay with the fresh validator.

`rotateToken: true` returns a new `managementToken` and `managementUrl` **once** and invalidates the
previous token immediately. Write the new one down before you acknowledge the response.

## 4. Delete — `deleteDynamicQr`

`DELETE /api/dynamic-qr/{slug}` with the bearer token, `If-Match` and `X-Dynamic-QR-Version`. Returns
`204`.

This is permanent. Every printed copy of that QR code starts returning `404` at `/r/{slug}` and there
is no undo. If you only want to stop traffic, `PATCH` `status: "paused"` instead — a paused record
returns `410`, which is a truthful "this was here and is gone" rather than "this never existed".

## Failure modes

| Status | Meaning | What to do |
|---|---|---|
| `400` | Non-HTTPS destination, unsupported field, bad label | Fix and resend |
| `401` | Missing/malformed/invalid capability token | Do not retry — the token is wrong or rotated |
| `403` | Cross-origin management write | Call server-side, not from a browser on another origin |
| `409` | Vault not empty (vault delete only) | Delete children first |
| `412` | Missing or stale `If-Match` | Re-`GET`, take the fresh validator, replay |
| `423` | Operator-blocked record | Not recoverable by the capability holder; contact support |
| `429` | Management or create rate limit | Back off for `Retry-After` |
| `503` | Storage unavailable, or 100,000-record capacity reached | Safe to retry — the spec states no record was changed |

## Never do this

Never put a `managementUrl` or `managementToken` into QR artwork, a shared manifest, a chat response
or a log line. The campaign workflow returns them in a private manifest precisely because they are
unrecoverable secrets. Treat them like a private key.
