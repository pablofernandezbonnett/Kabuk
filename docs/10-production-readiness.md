# 10. Production Readiness

## Goal

Production monitoring should answer two practical questions: can an agency search and create a reservation now, and where should an engineer investigate if it cannot? The first version does not need a complete observability platform or a named vendor. It needs a small set of useful signals.

## Mental model

The agency sees an API response. Behind that response, the API Gateway, service, database, and infrastructure may each add delay or fail. Monitoring follows that path from user impact to likely cause.

- **Metrics** are aggregated numbers, such as request count or database connections. They support dashboards and alerts.
- **Logs** record the detail of an individual request or error. They support investigation.
- **Traces** connect one request with its service and database work. They help find where a slow request spent its time.

## Weak approach vs better approach

**Weak approach:** collect every available metric and alert on every error.

**Better approach:** monitor signals that show agency impact first, then enough dependency signals to find the cause. A valid `409 Conflict` for sold-out inventory is useful information, but it is not automatically a production incident. A sustained `5xx` rate or an unavailable database is more urgent.

## Important signals

### API performance and gateway protection

| Signal | Why it matters |
| --- | --- |
| Response time by endpoint | Compare normal availability searches, which are around 250 ms, with sustained slow behaviour. A single slow request is less important than many requests taking longer than expected. |
| Request count and active requests | Shows traffic growth, unusual bursts, and whether the service has enough capacity. |
| Status-code rate by endpoint | Separates successful requests from client errors, conflicts, rate limits, and server failures. |
| Gateway response time and failures | Confirms that the public entry point is working before the request reaches the service. |
| Rate-limit rejections (`429`) and blocked requests | Shows whether the gateway protection is active and whether an agency or endpoint is being abused. |

The gateway applies request and rate limits before expensive service or database work. Platform edge protection handles network-level DDoS and reports unusual traffic and blocked requests.

### Errors and request investigation

| Signal | Why it matters |
| --- | --- |
| `5xx` responses and timeouts | Unexpected server or dependency failures. A sustained increase should alert an engineer. |
| `400`, `401`, and `403` trends | May show malformed agency requests, expired tokens, or an incorrect client integration. They are usually investigated as trends rather than immediate incidents. |
| `404` and `409` trends | Can reveal stale client data or increased competition for the same inventory. |
| Error logs with `requestId`, endpoint, status, and duration | Allow support and engineers to find the request without guessing. |

Logs must not contain JWT values, complete idempotency keys, full request bodies, stack traces returned to the client, or sensitive data. The error response can expose a safe `requestId` for support, as described in [Part 1](01-architecture.md#api-style-and-http-contract).

### Database

| Signal | Why it matters |
| --- | --- |
| Slow query duration and query count | Helps identify whether availability search or reservation work is taking too long. |
| Connection-pool use and waiting time | Shows whether requests are waiting for a database connection rather than executing. |
| Active connections, database locks, and waiting time | Separates a slow query from a query waiting for another transaction or disk work. |
| Database CPU, memory, disk I/O, and free storage | Shows resource pressure that can slow many queries or make the database unavailable. |
| Database availability and backup-job result | Detects an unavailable database or a recovery protection that is no longer succeeding. |

The investigation process for a slow availability query is described in [Part 8](08-performance-investigation.md).

### Reservations and business signals

| Signal | Why it matters |
| --- | --- |
| Reservation attempts and confirmed reservations | Shows whether agencies can complete the main business operation. |
| Insufficient-inventory conflicts (`409`) | A normal result in isolation, but a sudden increase can show strong demand, stale search data, or a client issue. |
| Unexpected reservation failures | A valid reservation request that repeatedly fails is an operational problem. |
| Idempotency replays and key/body conflicts | Replays confirm safe retries; conflicts can reveal a client bug that reuses a key for different data. |
| Inventory safety check | Negative availability must never occur. Seeing it is a serious data-integrity alert. |

These are aggregate operational signals. They can later support product analysis, such as understanding search-to-reservation behaviour, but this first version does not need to monitor personal data or full reservation details.

### Infrastructure

| Signal | Why it matters |
| --- | --- |
| Service health and readiness | Ensures the service can receive requests and reach its required dependencies. |
| CPU, memory, and restarts | Helps find resource exhaustion or an unstable service instance. |
| Network errors and service availability | Helps separate an application failure from a connectivity problem. |
| Number of running service nodes, when more than one exists | Confirms that enough nodes are healthy to serve traffic. |
| Node saturation and restarts, when more than one exists | Shows whether traffic is unevenly distributed or nodes need more capacity. |

The first design does not require several service nodes. If it later uses them for capacity or availability, these signals help avoid overload and unnecessary over-provisioning.

## Alerts and dashboards

I would alert for conditions that need prompt action: sustained `5xx` responses or timeouts, unavailable database or gateway, exhausted database connections, no healthy service node, or unexpected reservation failures.

I would use dashboards and regular review for trends that need investigation but are not necessarily incidents: `400`, `401`, `403`, `404`, `409`, `429`, searches with no availability, idempotency replays, and traffic growth.
