# Hotel Availability & Reservation Service

Technical assignment for a travel platform backend service. The work is organised in small Markdown documents so that each decision can be reviewed and explained on its own.

## Scope

Travel agencies can search for available hotel rooms and create reservations. The design focuses on correctness when availability changes at the same time as a reservation request.

## Design documents

- [1. Architecture](docs/01-architecture.md)
- [2. Hotel availability API](docs/02-availability-api.md)
- [3. Large search results](docs/03-large-search-results.md)
- [4. Reservation API](docs/04-reservation-api.md)
- [5. Concurrency](docs/05-concurrency.md)
- [6. Duplicate requests](docs/06-duplicate-requests.md)
- [7. Database design](docs/07-database-design.md)
- [8. Performance investigation](docs/08-performance-investigation.md)
- [9. Testing strategy](docs/09-testing-strategy.md)
- [10. Production readiness](docs/10-production-readiness.md)
- [11. Trade-offs](docs/11-trade-offs.md)

The optional implementation is intentionally not started. The written design is the priority for this assignment.

## Technology choice

- Java 21 and Spring Boot for the optional implementation.
- PostgreSQL as the source of truth for hotel, inventory, pricing, and reservation data.
- A REST API over HTTPS for travel agencies.

## AI tool usage

See [AI_USAGE.md](AI_USAGE.md).
