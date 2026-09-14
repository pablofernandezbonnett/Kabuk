# 11. Trade-offs

## Why this matters

This first version focuses on correct availability and reservations. The trade-offs make its scope and next priorities clear.

## What I would improve with another week

I would build and test a small PostgreSQL reservation flow for concurrency and idempotency. It would use real transactions for two requests for the last room and a retry after a lost response.

Concurrency and failure behaviour must be tested in the real components that enforce them. I would validate these high-risk rules before adding product features.

## What is intentionally simple

One reservation has one hotel, one room type, one date range, and a `CONFIRMED` status. Payment, cancellation, and several room types are outside the first version.

The architecture has one service and one PostgreSQL database. It has no queue or live availability cache because agencies need an immediate reservation result. These limits keep the API, data model, and transaction easy to change when a real requirement appears.

## Assumptions

- The platform already provides authentication, and the API Gateway passes a trusted agency identity and permissions to this service.
- One reservation is for one hotel, one room type, and one date range, with a quantity of that room type.
- A search result is not a hold; every reservation checks current inventory again before confirmation.
- Availability and price data exist for each room type and stay date.
- Payment, cancellation, email, and multi-hotel cart flows are outside the first version.
- The platform destination catalogue provides stable IDs and display names.
- Hotel data includes official star categories. Hotel and destination coordinates are available when distance sorting is requested.

## Largest technical risk

The largest technical risk is many reservation attempts for popular rooms and dates. Row locks prevent overselling, but competing requests can wait and time out.

Short transactions, a consistent date-lock order, and no external calls while locks are held reduce this risk. PostgreSQL integration tests and production monitoring of lock waits and reservation failures validate these controls.

## Practical rule

> Keep the first version small, validate its highest-risk rules with real components, and add complexity only when product or traffic needs require it.
