---
name: Spade
description: Use when enriching card transactions, bank transfers, or universal transaction data with merchant details, categories, and location information. Also use when setting up action triggers for rewards, fraud prevention, or transaction controls, or when personalizing transaction categories for end users.
metadata:
    mintlify-proj: spade
    version: "1.0"
---

# Spade Skill

## Product summary

Spade is a transaction enrichment API that transforms raw card, transfer, and payment data into clean, high-fidelity merchant insights. It matches transactions to verified merchants in its database (covering 99.9% of US merchants), returning counterparty IDs, logos, websites, precise locations, industry classifications, and optional premium data like recurrence detection and risk signals. Agents use Spade to enrich transactions in real-time (sub-50ms) or in batch (up to 50,000 per request), register action triggers that fire custom logic on matched transactions, and personalize transaction categories for end users.

**Key files and endpoints:**
- Authentication: `X-Api-Key` header (required for all requests)
- Environments: `https://east.sandbox.spade.com` (sandbox), `https://east.api.spade.com` (production); also west coast variants
- Core enrichment: `/transactions/cards/enrich`, `/transactions/transfers/enrich`, `/transactions/universal/enrich`
- Batch enrichment: `/batches/transactions/cards/enrich` (up to 50,000 transactions)
- Action triggers: `/merchant-action-triggers`, `/category-action-triggers` (register rules that fire on matched transactions)
- Primary docs: https://docs.spade.com

## When to use

Reach for this skill when:
- **Enriching transactions:** A user needs to add merchant details, clean merchant names, identify categories, or get location data for card, transfer, or aggregated payment transactions
- **Real-time enrichment:** Transactions arrive one at a time and need enrichment in <50ms (e.g., authorization flow, transaction posting)
- **Batch enrichment:** Historical transactions or bulk imports need enrichment without latency constraints
- **Action triggers:** A user wants to register rules that automatically fire rewards, block transactions, or flag for review when transactions match specific merchants or categories
- **Category personalization:** End users need to recategorize transactions or create custom categories for budgeting/PFM use cases
- **Error correction:** A user believes an enrichment response is inaccurate and needs to report it for review

Do not use Spade for: authentication, account management, payment processing, or merchant onboarding.

## Quick reference

### Authentication & Environments

| Environment | Endpoint | Use case |
|---|---|---|
| Sandbox (East) | `https://east.sandbox.spade.com` | Development and testing |
| Sandbox (West) | `https://west.sandbox.spade.com` | Development (west coast) |
| Production (East) | `https://east.api.spade.com` | Live transactions |
| Production (West) | `https://west.api.spade.com` | Live transactions (west coast) |

All requests require `X-Api-Key: YOUR_API_KEY` header.

### Enrichment Endpoints

| Endpoint | Input | Use case | Latency |
|---|---|---|---|
| `/transactions/cards/enrich` | Parsed merchant data (name, city, region, country) | Single card transaction | <50ms |
| `/transactions/cards/enrich/parse` | Raw DE43 string | Single card transaction with unparsed data | <50ms |
| `/transactions/transfers/enrich` | Transfer details (counterparty, amount, date) | Single bank transfer | <50ms |
| `/transactions/universal/enrich` | Generic transaction data | Single aggregated/third-party transaction | <50ms |
| `/batches/transactions/cards/enrich` | Array of up to 50,000 card transactions | Bulk historical enrichment | Async (poll or webhook) |
| `/batches/transactions/transfers/enrich` | Array of up to 50,000 transfers | Bulk transfer enrichment | Async |
| `/batches/transactions/universal/enrich` | Array of up to 50,000 universal transactions | Bulk aggregated enrichment | Async |

### Required Fields (Card Enrichment)

| Field | Type | Notes |
|---|---|---|
| `merchantName` | string | Raw merchant name from issuer (e.g., "SQ*WMSUPERCENTER#582") |
| `userId` | string | Unique user identifier (no PII; max 512 chars) |
| `amount` | string | Transaction amount (negative for credits, positive for debits) |
| `currencyCode` | string | ISO 4217 code (e.g., "USD") |
| `occurredAt` | ISO 8601 | Transaction timestamp |
| `categoryCode` | string | MCC or other category code |
| `categoryType` | string | "MCC" or other category type |
| `location.city` | string | City (required) |
| `location.country` | string | 3-letter country code or 3-digit numeric code |
| `transactionId` | string | Unique transaction ID (for correlation) |

Optional but recommended: `location.region`, `location.postalCode`, `acquirerId`, `cardId`, `programId`.

### Action Trigger Scopes

| Scope | Endpoint | Applies to | Per-request limit | Total cap |
|---|---|---|---|---|
| Account | `/merchant-action-triggers` | All transactions | 100,000 | 300,000 |
| Program | `/programs/{programId}/merchant-action-triggers` | Transactions with matching `programId` | 100,000 | 300,000 |
| User | `/users/{userId}/merchant-action-triggers` | All cards for a user | 100 | 100 |
| Card | `/users/{userId}/cards/{cardId}/merchant-action-triggers` | Single card only | 100 | 100 |

## Decision guidance

### Real-time vs. batch enrichment

| Scenario | Use real-time | Use batch |
|---|---|---|
| Authorization flow, transaction posting, user-facing features | ✓ | |
| Historical data import, bulk backfill, no latency requirement | | ✓ |
| Small number of transactions (<100) | ✓ | |
| Large volume (1000+), no latency requirement | | ✓ |
| Need results immediately | ✓ | |
| Can wait hours/days for results | | ✓ |

### Parsed vs. unparsed card data

| Scenario | Use `/enrich` | Use `/enrich/parse` |
|---|---|---|
| Issuer provides parsed merchant name, city, region, country | ✓ | |
| Issuer provides raw DE43 string | | ✓ |
| Data is already split into fields | ✓ | |
| Data is a single concatenated string | | ✓ |

### Merchant trigger matching modes

| Mode | Use when | Example |
|---|---|---|
| Location-level (default) | Trigger applies to a specific physical location | Reward only at Costco on 5th Ave, not all Costcos |
| Corporation-level | Trigger applies to entire brand | Reward at any Starbucks location worldwide |

### PUT vs. PATCH for action triggers

| Operation | Use PUT | Use PATCH |
|---|---|---|
| Initial setup or full sync | ✓ | |
| Adding/removing a few triggers | | ✓ |
| Registering >100,000 triggers in batches | | ✓ (for all but first batch) |
| Replacing entire trigger list | ✓ | |

## Workflow

### Enrich a single transaction (real-time)

1. **Prepare the request:** Gather merchant name, location, amount, currency, category code, user ID, and transaction ID from your source system.
2. **Choose the endpoint:** Use `/transactions/cards/enrich` for parsed data or `/transactions/cards/enrich/parse` for raw DE43 strings.
3. **Make the request:** POST to the appropriate endpoint with required fields and `X-Api-Key` header.
4. **Parse the response:** Extract `enrichmentId`, `counterparty` (merchant details), `location` (address, lat/long), `display` (UI-friendly name/logo), and `actions` (if triggers matched).
5. **Store the enrichmentId:** Use it to correlate enrichments with your transaction records and for error reporting.
6. **Use the data:** Display merchant name/logo to users, apply category-based rules, trigger rewards if `actions` matched.

### Enrich transactions in batch

1. **Prepare the batch:** Collect up to 50,000 transactions with unique `transactionId` values. Send oldest transactions first (improves recurrence detection).
2. **Submit the batch:** POST to `/batches/transactions/cards/enrich` with `transactions` array and optional `callbackUrl`.
3. **Capture the batchId:** Store it immediately — you'll need it to retrieve results.
4. **Monitor progress:** Either poll `/batches/{batchId}` with exponential backoff or wait for webhook callback to `callbackUrl`.
5. **Retrieve results:** Once status is `completed`, GET `/batches/{batchId}/results` to fetch enriched transactions.
6. **Handle errors:** Check `statusCode` for each transaction; errors include field-level details.
7. **Correlate and store:** Use `transactionId` to match results back to your records.

### Register merchant action triggers

1. **Ensure feature is enabled:** Contact sales@spade.com if you don't have access to actions.
2. **Choose scope:** Decide whether triggers apply at account, program, user, or card level.
3. **Prepare trigger list:** Define trigger ID, merchant name, location (or website), and custom `action` JSON (e.g., `{"type": "REWARD", "rewardPercent": 5}`).
4. **Register triggers:** PUT to the appropriate scope endpoint with `merchantTriggers` array.
5. **Check status:** GET the same endpoint to verify status is `succeeded` before enriching transactions.
6. **Enrich transactions:** When you enrich a transaction, matched triggers appear in the `actions` array of the response.
7. **Apply actions:** Use the `action` data to trigger rewards, block, or flag transactions.

### Report an enrichment error

1. **Identify the issue:** Note the `enrichmentId` and describe what you believe is inaccurate.
2. **Submit the report:** POST to `/transactions/report` with `enrichmentId`, `errorDescription`, and optional boolean flags (`incorrectCounterparty`, `incorrectLocation`, etc.).
3. **Wait for review:** Spade's team reviews your report (no SLA provided).
4. **Receive correction:** If a correction is made, Spade sends the corrected enrichment to your callback URL with `X-Webhook-Token` header.
5. **Validate and update:** Verify the token matches your stored callback token, then update your records with the corrected data.

## Common gotchas

- **Missing required fields:** Requests fail silently if `merchantName`, `city`, `country`, `amount`, `currencyCode`, `occurredAt`, `categoryCode`, or `userId` are missing. Always validate before sending.
- **Duplicate transaction IDs in batch:** Batch requests fail if any `transactionId` is duplicated. Ensure uniqueness within each batch.
- **Triggers not active until status is `succeeded`:** Transactions enriched while trigger registration is `pending` or `running` will NOT include the new triggers. Always poll until `succeeded`.
- **PUT replaces entire trigger list:** Using PUT to update a few triggers will delete all others at that scope. Use PATCH for incremental updates.
- **Batch size limits:** User and card scopes are capped at 100 triggers per request; account and program scopes allow up to 100,000 per request but 300,000 total per scope.
- **One registration at a time per scope:** If a batch is `pending` or `running`, any other PUT/PATCH to the same scope returns 409 Conflict. Send batches sequentially.
- **Latency varies by geography:** East and West coast servers exist; use the closest one to your infrastructure. VPNs and short-lived connections add latency.
- **Counterparty ID is null on no match:** If Spade doesn't match a merchant in its database, `counterparty.id` is null but merchant name is still cleaned and returned.
- **Third parties are separate from counterparties:** Always check the `thirdParties` array (e.g., DoorDash, Square) even if no counterparty matched.
- **Batch results not guaranteed in order:** Use `transactionId` to correlate results, not array position.
- **Callback token is a shared secret:** Store it securely; never expose in client-side code or logs.
- **Deprecated merchant matching endpoints:** The `/merchant-matching` endpoints are deprecated; use merchant search or enrichment instead.

## Verification checklist

Before submitting work with Spade:

- [ ] API key is valid and passed in `X-Api-Key` header
- [ ] Correct environment URL is used (sandbox for dev, production for live)
- [ ] All required fields are present and non-empty (`merchantName`, `userId`, `amount`, `currencyCode`, `occurredAt`, `categoryCode`, `city`, `country`)
- [ ] Transaction IDs are unique within batch requests
- [ ] Batch size does not exceed 50,000 transactions
- [ ] Trigger registration status is `succeeded` before enriching transactions
- [ ] Trigger IDs are unique within each scope
- [ ] Callback URL is HTTPS and reachable
- [ ] Callback token is stored securely and validated on incoming requests
- [ ] Error responses are parsed and handled (check `statusCode` and `errors` fields)
- [ ] Enrichment responses include `enrichmentId` for correlation and error reporting
- [ ] Counterparty and third party data are both checked (not just counterparty)
- [ ] Match scores are reviewed for low-confidence matches (consider fallback logic)
- [ ] Latency is measured from the closest geographic region

## Resources

- **Comprehensive page listing:** https://docs.spade.com/llms.txt
- **API Reference:** https://docs.spade.com/api-reference/spec.yml (OpenAPI spec)
- **Card Enrichment Guide:** https://docs.spade.com/reference/card-enrichment-guide
- **Action Triggers Guide:** https://docs.spade.com/reference/action-triggers-guide
- **Understanding Enriched Data:** https://docs.spade.com/reference/understanding-enriched-data

---

> For additional documentation and navigation, see: https://docs.spade.com/llms.txt