# 10. Production Readiness

## Goal

Production monitoring should answer two questions: can an agency search and reserve now, and where should an engineer investigate if not? The first version needs a small set of useful signals, not a complete observability platform or named vendor.

Start with agency impact. A `409 Conflict` for sold-out inventory is useful information, but not automatically an incident. Sustained `5xx` responses or an unavailable database need faster action.

## Mental model

The agency sees an API response. The gateway, service, database, and infrastructure can each add delay or fail. Monitoring follows that path from user impact to likely cause.

- **Metrics** are aggregated numbers, such as request count or database connections. They support dashboards and alerts.
- **Logs** record the detail of an individual request or error. They support investigation.
- **Traces** connect one request with its service and database work. They help find where a slow request spent its time.

## Important signals

### API performance and gateway protection

| Signal | Why it matters |
| --- | --- |
| Response time by endpoint | Shows sustained slow availability searches against the normal 250 ms response. |
| Request count and active requests | Shows traffic growth, unusual bursts, and capacity needs. |
| Status-code rate by endpoint | Separates success, client errors, conflicts, rate limits, and server failures. |
| Gateway response time and failures | Confirms the public entry point works before the request reaches the service. |
| Rate-limit rejections (`429`) and blocked requests | Shows whether protection is active and whether an agency or endpoint is being abused. |

The gateway applies request and rate limits before expensive service or database work. Platform edge protection handles network-level DDoS and reports unusual traffic and blocked requests.

### Errors and request investigation

| Signal | Why it matters |
| --- | --- |
| `5xx` responses and timeouts | Unexpected server or dependency failures. A sustained increase should alert an engineer. |
| `400`, `401`, and `403` trends | May show malformed requests, expired tokens, or an incorrect client integration. They are trends, not usually immediate incidents. |
| `404` and `409` trends | Can reveal stale client data or more competition for the same inventory. |
| Error logs with `requestId`, endpoint, status, and duration | Let support and engineers find the request without guessing. |

Logs must not contain JWT values, complete idempotency keys, full request bodies, stack traces returned to the client, or sensitive data. The error response can expose a safe `requestId` for support, as described in [Part 1](01-architecture.md#api-style-and-http-contract).

### Database

| Signal | Why it matters |
| --- | --- |
| Slow query duration and query count | Shows whether availability search or reservation work takes too long. |
| Connection-pool use and waiting time | Shows requests waiting for a database connection. |
| Active connections, database locks, and waiting time | Separates a slow query from one waiting for a transaction or disk work. |
| Database CPU, memory, disk I/O, and free storage | Shows resource pressure that can slow queries or make the database unavailable. |
| Database availability and backup-job result | Detects an unavailable database or failed recovery protection. |

The investigation process for a slow availability query is described in [Part 8](08-performance-investigation.md).

### Reservations and business signals

| Signal | Why it matters |
| --- | --- |
| Reservation attempts and confirmed reservations | Shows whether agencies can complete the main operation. |
| Insufficient-inventory conflicts (`409`) | Normal in isolation; a sudden increase can show demand, stale search data, or a client issue. |
| Unexpected reservation failures | Repeated failures for valid requests are an operational problem. |
| Idempotency replays and key/body conflicts | Replays confirm safe retries; conflicts can reveal key reuse with different data. |
| Inventory safety check | Negative availability must never occur; it is a serious data-integrity alert. |

These aggregate signals can later support product analysis, such as search-to-reservation behaviour. This first version does not need personal data or full reservation details.

### Infrastructure

| Signal | Why it matters |
| --- | --- |
| Service health and readiness | Ensures the service can receive requests and reach required dependencies. |
| CPU, memory, and restarts | Helps find resource exhaustion or an unstable service instance. |
| Network errors and service availability | Separates an application failure from a connectivity problem. |
| Healthy service nodes, when more than one exists | Confirms enough nodes can serve traffic. |
| Node saturation and restarts, when more than one exists | Shows uneven traffic or a need for more capacity. |

The first design does not require several service nodes. If it later uses them, these signals help avoid overload and adding capacity too early.

## Alerts and dashboards

Alert for conditions needing prompt action: sustained `5xx` responses or timeouts, unavailable database or gateway, exhausted database connections, no healthy service node, or unexpected reservation failures.

Use dashboards and regular review for trends that need investigation but are not necessarily incidents: `400`, `401`, `403`, `404`, `409`, `429`, searches with no availability, idempotency replays, and traffic growth.
