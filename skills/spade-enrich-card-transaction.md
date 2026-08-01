---
name: Enrich a card transaction with Spade
description: Turn a raw card transaction (merchant string, amount, MCC, location) into clean, structured merchant and category data, and report back if an enrichment is wrong.
api: openapi/spade-openapi-original.yml
operations: [cardEnrich, cardEnrichParse, cardReport]
---

# Enrich a card transaction with Spade

Use this skill to enrich a single card (credit/debit) transaction in real time.

## Auth & environment
- Send the secret key in the `X-Api-Key` request header.
- Choose the region-nearest host to minimize latency: `https://east.api.spade.com` or `https://west.api.spade.com` (use the `*.sandbox.spade.com` hosts while testing — sandbox and production keys are different).

## Steps
1. **Enrich** — call `cardEnrich` (`POST /transactions/cards/enrich`). Include every field you have (the `region`, `acquirerId`, etc. are optional but materially improve match quality). If you have raw ISO 8583 DE43 data, use `cardEnrichParse` (`POST /transactions/cards/enrich/parse`) instead so `city`/`merchantName` can be omitted.
2. **Read the result** — the response returns the enriched merchant, category, channel, recurrence signals, and a match score. Consult `data-model/spade-data-model.yml` for the entity shapes and `reference/match-score-guide` for interpreting the score.
3. **Report corrections** — if an enrichment is wrong, call `cardReport` (`POST /transactions/report`) with the `enrichmentId` and an `errorDescription`. Supply a `callbackUrl` to receive the corrected enrichment asynchronously (see `asyncapi/spade-webhooks.yml`).

## Error handling
- `400` bad request, `403` forbidden (check the API key/environment), `404` not found, `409` conflict, `429` rate limited, `500` server error. Responses are `application/json` (not RFC 9457). See `errors/spade-problem-types.yml`.
