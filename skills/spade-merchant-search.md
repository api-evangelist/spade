---
name: Search Spade's merchant database
description: Query Spade's merchant/corporation database to power autocomplete and merchant-lookup experiences.
api: openapi/spade-openapi-original.yml
operations: [corporationSearch]
---

# Search Spade's merchant database

Use this skill to look up merchants directly (e.g. to power an autocomplete UI), independent of a specific transaction.

> This endpoint is in beta; request access via sales@spade.com.

## Auth & environment
- `X-Api-Key` header; region-nearest host (`east`/`west`, `api`/`sandbox`).

## Steps
1. **Search** — call `corporationSearch` (`GET /corporations`) with your query terms. The response returns matching merchants from Spade's database suitable for autocomplete or enrichment previews (see `reference/merchant-search-guide`).

## Error handling
See `errors/spade-problem-types.yml`. Expect `403` if your key lacks merchant-search (beta) access.
