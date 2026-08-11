---
name: Generate a static QR code asset
description: >-
  Produce a single branded static QR file (SVG, PNG, JPG, WebP, PDF or EPS) from any of 27 payload
  types using QR Code Crafter, choosing the right format and error-correction level and staying inside
  the cost-weighted rate budget.
api: openapi/qr-code-crafter-openapi-original.json
operations:
  - generateQr
  - downloadQr
generated: '2026-08-11'
method: generated
source: >-
  Grounded in openapi/qr-code-crafter-openapi-original.json (operationIds verified verbatim),
  conventions/qr-code-crafter-conventions.yml, errors/qr-code-crafter-problem-types.yml and
  rate-limits/qr-code-crafter-rate-limits.yml.
---

# Generate a static QR code asset

A static QR code encodes its payload directly. Once it is printed it cannot be changed. If the
destination may ever need to move, do not use this skill — use `create-dynamic-qr` instead.

## Before you call

No credential is required. If you were issued one, send it as `X-Agent-Api-Key`; it raises the rate
budget from 180 to 600 cost units per 60-second window. Never invent a key — an unauthenticated call
works unless the deployment has enabled key enforcement, in which case you get `401` with a
`WWW-Authenticate` challenge.

## Choose the shape of the call

Two operations hit the same path and both are correct:

- **`generateQr`** — `POST /.netlify/functions/generate-qr` with a JSON body. Use this for anything
  long or awkward: vCards, calendar events, Wi-Fi credentials, payment payloads, logo data URLs. The
  response carries `dataUrl` for a preview and base64 `data` for persistence, plus `byteLength`,
  `dimensions` and the normalized `qrOptions`.
- **`downloadQr`** — `GET /.netlify/functions/generate-qr?value=...&format=...`. Use this when you
  want raw bytes and nothing else. The response headers `X-QR-Width`, `X-QR-Height`, `X-QR-Unit` and
  `X-QR-Bytes` let you verify the saved file without parsing it.

Prefer POST for automation. Reserved URL characters in a query string are the most common cause of a
`400` here.

## Build the payload

Do not guess the payload encoding. The provider publishes 27 ready-to-send request bodies — one per
payload type — as `x-agent-recipes` on the `GenerateQrRequest` schema, mirrored in
`examples/qr-code-crafter-agent-recipes.yml`. Start from the recipe whose key matches the payload type
(`url`, `wifi`, `vcard`, `event`, `upi`, `sepa`, `swissqr`, …) and substitute the real values. Each
recipe also names `bestFormats` for that type.

Request fields: `value` (required), `format`, `size` (256–4096), `margin` (0–16), `fgColor`,
`bgColor`, `errorCorrectionLevel` (`L`/`M`/`Q`/`H`), and `logo` as a base64 PNG/JPG/WebP data URL.

## Choose format and error correction

- Print, packaging, signage → `svg`, `pdf` or `eps` (vector, no compression artifacts).
- Web, email, documents → `png`, `svg` or `webp`.
- Embedding a logo → raise `errorCorrectionLevel` to `H`. A logo occludes modules; `H` is what buys
  back the redundancy.
- High volume → `svg`. It costs 1 budget unit against `png` 3, `jpg`/`webp` 4 and `pdf` 6.

## Stay inside the budget

The budget is cost-weighted, not request-counted, and the costs compose: a 3072px PDF with a logo
bills 6 + 4 + 4 = 14 of your 180 units per minute. Pace accordingly. On `429` or `503`, read
`Retry-After` and back off; read `X-RateLimit-Limit` and `X-RateLimit-Remaining` when present. Do not
retry immediately — these headers appear only under pressure, so a tight retry loop will simply burn
the next window too.

## Handle failures

Errors are flat JSON, `{"success": false, "error": "..."}` — there is no machine-readable error code,
so branch on the HTTP status:

| Status | Meaning | What to do |
|---|---|---|
| `400` | Invalid value, format, size or margin | Fix the request; do not retry unchanged |
| `401` | Key enforcement is on | Retry with `X-Agent-Api-Key` |
| `405` | Wrong method for the path | Use GET or POST as above |
| `429` | Budget exhausted | Back off for `Retry-After`, then retry |
| `500` | Generation pipeline error | Retry once with backoff |

## Verify before you publish

Generation success is not readability. If the asset is going to print or into a product, use the
`generate-and-verify` skill instead — it decodes the finished file and compares payload hashes.
