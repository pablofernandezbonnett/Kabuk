# 11. Trade-offs

## Why this matters

This first version focuses on correct availability and reservation behaviour. The trade-offs below make its current scope clear and identify the next work that would add the most confidence.

## What I would improve with another week

I would build and test a small PostgreSQL-based reservation vertical slice for concurrency and idempotency. The goal would be to validate the current design with real transactions, including two requests for the last room and a retry after a lost response.

This is not because the design is assumed to be wrong. Database concurrency and failure behaviour must be tested in the real components that enforce them. I would validate the highest-risk rules before adding more product features.

## What is intentionally simple

The reservation model supports one hotel, one room type, one date range, and a `CONFIRMED` status. It does not include payment, cancellation, or several room types in one reservation.

The architecture also keeps one service and one PostgreSQL database. It does not add a queue or a live availability cache because an agency needs an immediate, correct reservation result. These limits keep the API, data model, and transaction easy to explain and change when a real new requirement appears.

## Assumptions

- The platform already provides authentication, and the API Gateway passes a trusted agency identity and permissions to this service.
- One reservation is for one hotel, one room type, and one date range, with a quantity of that room type.
- A search result is not a hold; every reservation checks current inventory again before confirmation.
- Availability and price data exist for each room type and stay date.
- Payment, cancellation, email, and multi-hotel cart flows are outside the first version.
- The platform destination catalogue provides stable IDs and display names.
- Hotel data includes official star categories. Hotel and destination coordinates are available when distance sorting is requested.

## Largest technical risk

The largest technical risk is high contention for popular room types and dates. Row-level locks prevent overselling, but many reservation attempts for the same inventory can wait and may time out.

The design reduces this risk with short transactions, a consistent date-lock order, and no external calls while locks are held. Real PostgreSQL integration tests and production monitoring of lock waits and reservation failures are needed to validate those controls.

## Practical rule

> Keep the first version small, validate its highest-risk correctness rules with real components, and extend it only when product or traffic needs justify the added complexity.
