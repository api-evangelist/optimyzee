---
name: optimyzee-audit-and-optimize-account
description: Audit a linked Google Ads account, score its health, clean wasted search terms, and apply the resulting optimization tasks — the full Optimyzee analyze-then-act loop.
api: Optimyzee Application API
generated: '2026-08-12'
method: generated
source: openapi/_original/optimyzee-openapi.json (harvested from https://api.optimyzee.com/docs)
operations:
  - appLinkingGoogleAdsPing
  - appLookupGoogleAdsCampaignQuery
  - appCampaignGoogleAdsCreate
  - appAnalyzeGoogleAdsAuditQueue
  - appAnalyzeGoogleAdsAuditView
  - appAnalyzeGoogleAdsHealthAnalyze
  - appAnalyzeGoogleAdsOptimizationAnalyze
  - appAnalyzeGoogleAdsOptimizationView
  - appAnalyzeGoogleAdsOptimizationMarkResult
  - appAnalyzeGoogleAdsOptimizationHistory
  - appAnalyzeGoogleAdsOptimizationRecommendations
  - appCampaignActivityQuery
  - appCampaignActivityApply
  - appCampaignActivityDismiss
  - appCampaignGoogleAdsMetrics
---

# Audit an account, then act on what it finds

## Preconditions

- Bearer token in `authorization`; base `https://api.optimyzee.com`.
- A `linkingId` (see `optimyzee-link-google-ads-account`). Confirm it is alive with
  `GET /app/linking/g/{linkingId}/ping` (`appLinkingGoogleAdsPing`) before queueing anything.

## Steps

1. **Find the campaign** — `GET /app/lookup/g/campaign` (`appLookupGoogleAdsCampaignQuery`).
   Filters include `linkingId`, `advertisingChannelType`, `ids`, `resourceNames`, `q`.
2. **Track it in Optimyzee** — `POST /app/campaign/googleAds` (`appCampaignGoogleAdsCreate`) with
   `{ "linkingId": …, "campaignId": … }`.
3. **Score the account** — `POST /app/analyze/g/health` (`appAnalyzeGoogleAdsHealthAnalyze`).
   Required: `linking_id`. Optional: `period_days`. Returns the account/campaign health report with
   metrics, deltas and flags.
4. **Queue an audit** — `POST /app/analyze/g/audit` (`appAnalyzeGoogleAdsAuditQueue`) with
   `{ "linkingId": …, "campaignId": …, "scope": … }`. `scope` is an enum:
   `CAMPAIGN`, `SEARCH_ADS`, `SEARCH_KEYWORDS`, `SEARCH_TERMS`, `P_MAX`. Returns `{ "id": … }`.
5. **Read the audit** — `GET /app/analyze/g/audit/{auditId}` (`appAnalyzeGoogleAdsAuditView`). Poll;
   audits are queued work.
6. **Clean wasted search terms** — `POST /app/analyze/g/optimization/search-terms`
   (`appAnalyzeGoogleAdsOptimizationAnalyze`) with `{ "linking_id": …, "campaign_id": … }`, then
   `GET /app/analyze/g/optimization/search-terms/{id}` (`appAnalyzeGoogleAdsOptimizationView`) for
   the results.
7. **Decide on each result** — `POST /app/analyze/g/optimization/search-terms/results/mark`
   (`appAnalyzeGoogleAdsOptimizationMarkResult`) with
   `{ "analysis_id": …, "result_ids": […], "action": …, "optimized": … }`.
   `analysis_id`, `result_ids` and `action` are required.
8. **Work the task queue** — `GET /app/campaign/activity/{campaignId}` (`appCampaignActivityQuery`)
   lists generated tasks; `POST /app/campaign/activity/apply/{taskId}`
   (`appCampaignActivityApply`) applies one, `POST /app/campaign/activity/dismiss/{taskId}`
   (`appCampaignActivityDismiss`) rejects it.
9. **Verify the effect** — `GET /app/campaign/googleAds/{campaignId}/metrics`
   (`appCampaignGoogleAdsMetrics`) and
   `GET /app/analyze/g/optimization/optimized-history` (`appAnalyzeGoogleAdsOptimizationHistory`).

## Conventions that bite

- **Field casing is inconsistent across this API.** The audit body uses `linkingId`/`campaignId`;
  the search-terms and health bodies use `linking_id`/`campaign_id`. Read the schema per operation.
- **Pagination is cursor-based**: `cursor` + `perPage` query params, with
  `x-pagination-next`, `x-pagination-previous`, `x-pagination-per-page`, `x-pagination-last` and
  `x-pagination-total` response headers. Those headers are exposed over CORS but are **not declared
  in the spec** — read them off the response, do not expect the contract to describe them.

## Error handling

- `422` → `{ "errors": { "<field>": ["message"] } }`; `400`/`401`/`403` →
  `{ "error": …, "message": … }`. No `429`, no `5xx` are modelled.

## Cautions

- Steps 7 and 8 **change a live advertising account** — applying a task or marking a search term
  optimized writes negative keywords and structural changes into Google Ads. Confirm with a human.
- There is no idempotency key on this API and no `Retry-After`. Do not blind-retry a `POST`; re-read
  state with the matching `…View` / `…Query` operation first.
