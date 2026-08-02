---
name: Create and track an SPV request
description: Submit a contract-validated SPV creation request idempotently, then poll or cancel it.
api: openapi/zest-equity-openapi-original.json
operations: [list_templates_v1_contracts_templates__version__get, createSpvRequest, getSpvRequest, listSpvRequests, cancelSpvRequest]
---

# Create and track an SPV request

Submit an SPV creation request that Zest admins review; on approval the SPV is materialised and a `spv_request.completed` webhook fires.

## Prerequisites
- A valid bearer access token (see `zest-equity-authenticate`).
- A `templateId` + `templateVersion` assigned during onboarding.

## Steps
1. (Optional) Discover the contract shape with **`list_templates_v1_contracts_templates__version__get`** — `GET /v1/contracts/templates/{version}` — to see which attributes the template requires.
2. Call **`createSpvRequest`** — `POST /v1/spv-requests`. The `Idempotency-Key` header is **required** here (missing → `400 invalid_request`); use a v4 UUID. Body carries `templateId`, `templateVersion`, and the `attributes` object validated against the template. Success returns `201` with `{ spvRequestSlug (svr_...), status: "pending-review", ... }` and fires `spv_request.created`.
3. Track state with **`getSpvRequest`** — `GET /v1/spv-requests/{slug}` — or **`listSpvRequests`** — `GET /v1/spv-requests?page=&perPage=&status=` (paginated, scoped to your `client_id`).
4. To withdraw a still-pending request, call **`cancelSpvRequest`** — `DELETE /v1/spv-requests/{slug}`. A terminal-state request (approved/rejected/already-cancelled) returns `409 conflict`.

## Conventions & failure rules
- Idempotency: same key + same body replays the cached response for 24h; same key + different body → `409 conflict`.
- `400 validation_error`: inspect `validationErrors[]` (per-field `code` such as `required`, `enum`, `min_value`).
- `403 forbidden`: token valid but partner not provisioned for the resource.
- Bodies are JSON, camelCase.
