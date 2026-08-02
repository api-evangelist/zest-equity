---
name: Authenticate with the Zest Public API
description: Mint a short-lived bearer access token using the OAuth 2.0 JWT-Bearer assertion grant, then verify it.
api: openapi/zest-equity-openapi-original.json
operations: [exchangeJwtAssertion, getOauth2Info]
---

# Authenticate with the Zest Public API

Every Zest partner call needs a bearer access token. Tokens are minted from a self-signed JWT assertion (RFC 7523), never issued directly.

## Prerequisites
- A `client_id` and the matching **EdDSA (Ed25519)** private key registered with Zest (provisioned at onboarding via `sara@zestholdco.com`).
- The environment base URL: `https://sandbox-api.zestequity.com` (sandbox) or `https://public-api.zestequity.com` (production).

## Steps
1. Build a JWT assertion signed with **EdDSA** (RSA/HMAC are rejected). Claims:
   - `iss` = your API base URL, `sub` = your `client_id`, `client_id` = same value (must equal `sub`).
   - `aud` = the **exact** environment base URL you are calling.
   - `iat` = now, `exp` = now + 60 seconds.
2. Call **`exchangeJwtAssertion`** — `POST /v1/oauth2/tokens` with `Content-Type: application/x-www-form-urlencoded` and body `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer&assertion=<JWT>`. Response: `{ accessToken, refreshToken, tokenType, expiresIn (3600), refreshExpiresIn (2592000) }`.
3. Send `Authorization: Bearer <accessToken>` on every subsequent call.
4. Optionally confirm the token with **`getOauth2Info`** — `GET /v1/oauth2/info` — which returns client type, client id, and application name.
5. Refresh proactively (at ~half of `expiresIn`, or on the first `401`) using `grant_type=refresh_token&refresh_token=<token>`. **Refresh tokens rotate on every use** — store the new one immediately; redeeming a stale refresh token returns `401` and invalidates the whole chain, forcing a fresh JWT assertion.

## Failure rules
- `401 invalid_token`: clock skew (enable NTP), `aud` mismatch, non-EdDSA algorithm, or `sub` != `client_id`.
- Parse the error `code` first; cite `errorId` to support.
