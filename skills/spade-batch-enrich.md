---
name: Batch-enrich transactions with Spade
description: Submit many card transactions for enrichment as a batch job, then poll for status and retrieve results (or receive a completion callback).
api: openapi/spade-openapi-original.yml
operations: [batchesTransactionsCardsEnrich, batchesCardEnrichGetStatus, batchesCardEnrichGetResults]
---

# Batch-enrich transactions with Spade

Use this skill to enrich a large set of card transactions asynchronously.

## Auth & environment
- `X-Api-Key` header; region-nearest host (`east`/`west`, `api`/`sandbox`).

## Steps
1. **Submit the batch** — call `batchesTransactionsCardsEnrich` (`POST /batches/transactions/cards/enrich`). The response returns a `batchId`. For small datasets you can pass `?synchronous=true` to get results inline (the per-request cap is `synchronousMax`, readable from the endpoint metadata/OPTIONS operation).
2. **Optionally register a callback** — include a `callbackUrl` in the request body and Spade will `POST` to it when the job finishes, with a verification token to confirm the sender. See `asyncapi/spade-webhooks.yml`.
3. **Poll status** — if not using a callback, call `batchesCardEnrichGetStatus` (`GET /batches/{cardEnrichmentBatchId}`) until the job is complete.
4. **Fetch results** — call `batchesCardEnrichGetResults` (`GET /batches/{cardEnrichmentBatchId}/results`) to retrieve the enriched records.

Transfer and universal batches follow the same submit -> status -> results pattern with their own enrich operations.

## Error handling
See `errors/spade-problem-types.yml`. A `409` means a conflicting job/registration is already in progress for the scope.
