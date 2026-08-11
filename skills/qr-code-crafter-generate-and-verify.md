---
name: Generate and verify a production QR candidate
description: >-
  Produce a final QR asset and prove it scans — QR Code Crafter generates the file, reviews the payload,
  assesses quiet zone and contrast, decodes the finished bytes and compares payload hashes, returning an
  evidence receipt without echoing the decoded payload.
api: openapi/qr-code-crafter-openapi-original.json
operations:
  - generateVerifiedQr
generated: '2026-08-11'
method: generated
source: >-
  Grounded in openapi/qr-code-crafter-openapi-original.json (operationId verified verbatim) and
  errors/qr-code-crafter-problem-types.yml.
---

# Generate and verify a production QR candidate

Use this before anything goes to print, packaging, signage or a product surface. Ordinary generation
tells you a file was produced; this tells you the file decodes back to what you asked for.

## Call it

`POST /.netlify/functions/generate-verified-qr` — operation `generateVerifiedQr`.

The request is the same shape as ordinary generation, with one hard constraint: **only `svg`, `png`,
`jpg` and `webp` can be verified.** `pdf` and `eps` cannot be decode-checked and the call returns
`400` if you ask for them. Generate the vector master separately with `generateQr` and verify the
raster twin.

## What comes back

A `QrVerificationReceipt`:

- `status` and `productionReady` — the headline verdict.
- `payload` (`QrSafePayloadReview`) — the structural review of what you asked to encode.
- `productionChecks` (`QrProductionChecks`) — bounded settings assessment: quiet zone, contrast,
  error correction, size.
- `decode` — the result of rasterizing and decoding the generated file and comparing payload hashes.
- `cautions` — non-fatal warnings worth surfacing to a human.
- `evidence` — links backing the verdict.
- `limitation` — the provider's own scope disclaimer, carried inside the payload.

The receipt deliberately **does not echo the decoded payload content**. Do not treat its absence as a
failure, and do not try to reconstruct it.

## Read the verdict honestly

`limitation` exists because this is a decode-and-settings check, not a certification. The provider
states plainly that it is *"not print, device, ISO, accessibility, malware, or security
certification."* Report `productionReady` as what it is — evidence that these bytes decoded on this
decoder — and never as a compliance claim.

## Failure modes

| Status | Meaning | What to do |
|---|---|---|
| `400` | Invalid request, or a non-decodable format (`pdf`/`eps`) | Switch to `svg`/`png`/`jpg`/`webp` |
| `422` | Blocked before generation, **or** the final asset failed decode / hash comparison | Two different schemas back this status — read which one you got before acting |
| `429` | Cost budget or verification concurrency limit reached | Back off for `Retry-After`; verification is more expensive than plain generation |
| `500` | Generation or verification pipeline error | Retry once with backoff |

A `422` is the interesting one. `GenerateVerifiedQrBlockedResponse` means the payload never should
have been encoded and nothing was generated. `GenerateVerifiedQrFailureResponse` means a file was
produced and then failed its own decode — surface that to a human rather than retrying, because a
retry will usually reproduce it.

## Then do the physical check

A passing receipt is a floor, not a ceiling. Scan the downloaded file, the placed layout, the exported
PDF and a physical proof at realistic distances, angles and lighting before release.
