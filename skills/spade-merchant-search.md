---
name: Search Spade's merchant database
description: Query Spade's merchant/corporation database to power autocomplete and merchant-lookup experiences.
api: openapi/spade-openapi-original.yml
operations: [corporationSearch, merchantSearch]
---

# Search Spade's merchant database

Use this skill to look up merchants directly (e.g. to power an autocomplete UI), independent of a specific transaction.

> Both endpoints are in beta; request access via sales@spade.com.

## Auth & environment
- `X-Api-Key` header; region-nearest host (`east`/`west`, `api`/`sandbox`).

## Steps
1. **Search corporations** — call `corporationSearch` (`GET /corporations`) with your query terms. The response returns matching merchants from Spade's database suitable for autocomplete or enrichment previews (see `reference/merchant-search-guide`).
2. **Get detailed merchant information (beta)** — call `merchantSearch` (`GET /merchants`) with the required `name` query parameter to retrieve a full `MerchantSearchResponse`: the same merchant records that appear as counterparties and third parties in enrichment responses, including industry categories, MCCs and affiliates. Whether a merchant surfaces as a counterparty or a third party in a given enrichment depends on the role it plays in that transaction.

## Caveats
- `GET /merchants` currently covers only a subset of Spade's merchants — mostly larger, frequently transacted ones — with more added over time.
- Merchant `id`s are "relatively stable" but can shift from mergers/acquisitions and from database improvements. Do not treat a merchant id as a permanent primary key.
- `GET /merchants` is marked `x-excluded: true` in Spade's published OpenAPI, so it is present in the machine-readable contract but hidden from the rendered reference.

## Error handling
See `errors/spade-problem-types.yml`. Expect `403` if your key lacks merchant-search (beta) access.
