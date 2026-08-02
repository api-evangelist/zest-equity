---
name: Onboard investors and record a subscription
description: Bulk-create investors, create a subscription on an SPV, then upload the signed form and funding receipt in strict order.
api: openapi/zest-equity-openapi-original.json
operations: [bulkCreateInvestors, createSubscriptions, uploadSubscriptionForm, uploadFundingReceipt]
---

# Onboard investors and record a subscription

The partner-managed subscription flow: create investors, subscribe them to an SPV, then attach the signed form and wire receipt — in that order.

## Prerequisites
- A valid bearer access token (see `zest-equity-authenticate`).
- A materialised SPV `spv_slug` (from an approved SPV request).

## Steps
1. Call **`bulkCreateInvestors`** — `POST /v1/investors`. Always returns `200` with a per-row `{ status, zestPersonId | error }` keyed by `partnerInvestorId` (partial success — bad rows do not fail the batch). Replaying the same `partnerInvestorId` returns the existing `zestPersonId`, so bulk loads are crash-safe. `Idempotency-Key` is optional. Fires `investor.created` per created row.
2. Call **`createSubscriptions`** — `POST /v1/spvs/{spv_slug}/subscriptions` — with one or more rows (per investor: lump sum + share class). `Idempotency-Key` optional. Fires `subscription.created`. Returns `201`.
3. Upload the signed subscription form with **`uploadSubscriptionForm`** — `POST /v1/spvs/{spv_slug}/subscription/{person_id}/forms`. `multipart/form-data`, single `file` field, ≤10 MB, content types PDF/JPEG/PNG/WEBP. Fires `signed_subscription_form.uploaded`.
4. Upload the wire receipt with **`uploadFundingReceipt`** — `POST /v1/spvs/{spv_slug}/subscription/{person_id}/fundings` — with `amount`/`currency`/`wire_reference`. **Strict order: the signed form must already be on record, else `409 conflict`.** Fires `funding_receipt.uploaded`.
5. Zest admin transition to Completed fires `subscription.completed`.

## Failure rules
- `413 payload_too_large` (>10 MB), `415 unsupported_media_type` (bad content type) on uploads.
- `409 conflict` on strict-order violation (funding before form) or idempotency-key body mismatch.
- Uploads are content-addressed, so `Idempotency-Key` is not used on the form/funding endpoints.
