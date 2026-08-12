---
name: optimyzee-build-ai-search-campaign
description: Build a Google Ads search campaign from a website URL with Optimyzee's AI pipeline — keyword plan, STAG structure, RSA copy — and publish it to the linked Google Ads account.
api: Optimyzee Application API
generated: '2026-08-12'
method: generated
source: openapi/_original/optimyzee-openapi.json (harvested from https://api.optimyzee.com/docs)
operations:
  - appCreateGoogleAdsSearchCreationAiCreate
  - appCreateGoogleAdsSearchCreationAiStatus
  - appCreateGoogleAdsSearchCreationAiBuildKeywordPlanner
  - appCreateGoogleAdsSearchCreationAiBuildStagStructure
  - appCreateGoogleAdsSearchCreationAiBuildRsaAds
  - appCreateGoogleAdsSearchCreationAiView
  - appCreateGoogleAdsSearchCreationCreate
  - appCreateGoogleAdsSearchCreationUpdate
  - appCreateGoogleAdsSearchCreationStatus
  - appCreateGoogleAdsSearchCreationPublish
---

# Build and publish an AI search campaign

This is Optimyzee's marquee flow: a website URL in, a structured, validated Google Ads search campaign
out. It is asynchronous at every stage — each build step queues work and you poll a status endpoint.

## Preconditions

- Bearer token in `authorization`; base `https://api.optimyzee.com`.
- A `linkingId` from `optimyzee-link-google-ads-account` if you intend to publish.

## Steps

1. **Start the AI draft** — `POST /app/create/g/searchCreationAi`
   (`appCreateGoogleAdsSearchCreationAiCreate`).
   Required: `url`. Optional: `linkingId`, `languages`, `locations`, `knownCompetitors`, `exclusions`.
   Returns the draft id.
2. **Poll** — `GET /app/create/g/searchCreationAi/{draftId}/status`
   (`appCreateGoogleAdsSearchCreationAiStatus`) until the analysis completes. There is no webhook and
   no event surface on this API, so polling is the only option.
3. **Build the keyword plan** — `POST /app/create/g/searchCreationAi/{draftId}/keywordPlanner`
   (`appCreateGoogleAdsSearchCreationAiBuildKeywordPlanner`).
   Required: `source`. Optional: `manualKeywords` when you want to seed your own.
4. **Build the STAG structure** — `POST /app/create/g/searchCreationAi/{draftId}/stag-builder`
   (`appCreateGoogleAdsSearchCreationAiBuildStagStructure`).
   Required: `keyword_ids`, `campaign_type`. Optional: `campaign_name`, `max_cpc`,
   `landing_page_url`. Note the snake_case body here — this API mixes camelCase and snake_case
   between operations, so read each request schema rather than assuming.
5. **Build the ads** — `POST /app/create/g/searchCreationAi/{draftId}/rsa-builder`
   (`appCreateGoogleAdsSearchCreationAiBuildRsaAds`). No body.
6. **Read the finished draft** — `GET /app/create/g/searchCreationAi/{draftId}`
   (`appCreateGoogleAdsSearchCreationAiView`).
7. **Promote to a publishable draft** — `POST /app/create/g/searchCreation`
   (`appCreateGoogleAdsSearchCreationCreate`). Required: `url`. Optional: `linkingId`,
   `rsaBuilderId`, `keywordPlannerId`, `languages`, `locations`, `budgetAmount`.
8. **Edit before publishing** — `PATCH /app/create/g/searchCreation/{draftId}`
   (`appCreateGoogleAdsSearchCreationUpdate`) to adjust `headlines`, `descriptions`, `keywords`,
   `budgetAmount`, `languages`, `locations`.
9. **Publish** — `POST /app/create/g/searchCreation/{draftId}/publish`
   (`appCreateGoogleAdsSearchCreationPublish`), then poll
   `GET /app/create/g/searchCreation/{draftId}/status`
   (`appCreateGoogleAdsSearchCreationStatus`).

## Error handling

- `422` returns `{ "errors": { "<field>": ["message"] } }`. On the STAG step this is where invalid
  `keyword_ids` or an unsupported `campaign_type` surface.
- `401` / `403` as in the linking skill.
- No `429` and no `5xx` are declared anywhere in this contract, so treat any non-declared status as
  unmodelled and stop rather than loop.

## Cautions

- **Step 9 spends money.** Publishing pushes a live campaign into the connected Google Ads account.
  Require explicit human confirmation before calling `appCreateGoogleAdsSearchCreationPublish`.
- **No idempotency key.** Do not retry any `POST` in this flow on a timeout. Re-read the draft with
  `appCreateGoogleAdsSearchCreationAiView` or `appCreateGoogleAdsSearchCreationView` and continue from
  observed state.
- Free-tier accounts are quota-capped; check `GET /app/campaign/free-limit-check`
  (`appFreeLimitCheck`) before starting a build you cannot finish.
