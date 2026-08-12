---
name: optimyzee-link-google-ads-account
description: Connect a Google Ads account to Optimyzee and confirm the link is healthy, so every other Optimyzee flow has a linkingId to work with.
api: Optimyzee Application API
generated: '2026-08-12'
method: generated
source: openapi/_original/optimyzee-openapi.json (harvested from https://api.optimyzee.com/docs)
operations:
  - appLinkingGoogleAdsUri
  - appLinkingGoogleAdsIssue
  - appLinkingGoogleAdsQuery
  - appLinkingGoogleAdsView
  - appLinkingGoogleAdsPing
  - appLinkingGoogleAdsHierarchy
  - appLinkingGoogleAdsDelete
---

# Link a Google Ads account

Nearly every Optimyzee operation takes a `linkingId`. Establish one first.

## Preconditions

- A bearer token in the `authorization` header. Tokens come from `/app/auth/gateway/*`
  (`appAuthGatewayEmail`, `appAuthGatewayGoogle`, …) and are rotated with `appAuthGatewayRefresh`.
  There is no self-issuable API key.
- Base URL `https://api.optimyzee.com`.

## Steps

1. **Get the consent URL** — `GET /app/linking/g/uri` (`appLinkingGoogleAdsUri`). Takes a
   `redirectUri` query parameter. Send the user through it; Google returns an authorization `code`.
2. **Exchange the code** — `POST /app/linking/g` (`appLinkingGoogleAdsIssue`) with
   `{ "code": "...", "redirectUri": "..." }`. Both fields are required. This creates the linking.
3. **List linkings** — `GET /app/linking/g` (`appLinkingGoogleAdsQuery`) to recover `linkingId`s.
   Supports the `isManager` query filter.
4. **Read the account tree** — `GET /app/linking/g/hierarchy` (`appLinkingGoogleAdsHierarchy`) when the
   link is an MCC/manager account, to find the child customer accounts.
5. **Check liveness before any long job** — `GET /app/linking/g/{linkingId}/ping`
   (`appLinkingGoogleAdsPing`). Do this before queueing an audit or an optimization; a stale Google
   token surfaces here rather than as a confusing failure three calls later.
6. **Remove a link** — `DELETE /app/linking/g/{linkingId}` (`appLinkingGoogleAdsDelete`).

Meta Ads and Yelp have the same shape under `/app/linking/meta/*` and `/app/yelp/connection`.

## Error handling

- `401` — token missing or expired. Refresh with `appAuthGatewayRefresh`, do not retry blindly.
- `403` — authenticated but not permitted for this linking.
- `422` — validation failure; the body is a field-keyed bag,
  `{ "errors": { "redirectUri": ["..."] } }`. Fix the named field; retrying unchanged will fail again.
- Every other error is `{ "error": "...", "message": null|"..." }`.

## Cautions

- **No idempotency key exists on this API.** `POST /app/linking/g` is not safe to blind-retry — a
  duplicate exchange may create a duplicate linking. On a timeout, call `appLinkingGoogleAdsQuery` and
  reconcile before retrying.
- No rate limit is published and no `429` is declared. Pace yourself; you will get no signal.
