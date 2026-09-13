# 6. Duplicate Requests

## Why this matters

A network timeout does not prove that a reservation failed. The service may have created it, but the agency may not have received the response. A retry must not create a second reservation or reduce inventory again.

## Mental model

An `Idempotency-Key` identifies one logical reservation operation for one agency. It is not a reservation ID and cannot be reused for a different reservation. A retry sends the same key and receives the first result again.

### Weak approach versus the selected approach

A weak approach creates a reservation for every request, so a timeout can create the same reservation twice. The selected approach records the key and result in PostgreSQL; a unique constraint makes same-key requests one logical operation.

## Idempotency records in PostgreSQL

Reservations and idempotency records have different responsibilities.

| Data | Purpose | Retention |
| --- | --- | --- |
| `reservations` | Business record of a confirmed reservation, including its agency, dates, price, and status. | Kept or archived under a separate business retention policy. |
| `idempotency_records` | Technical record that safely replays one API operation. It stores the agency, key, request fingerprint, result, and reservation reference. | Kept for a configurable 24-hour retry window. |

An idempotency record contains at least:

```text
agency_id
idempotency_key
request_fingerprint
reservation_id
response_status
response_body
created_at
expires_at
```

`reservation_id` can be empty for a business conflict. PostgreSQL enforces uniqueness on `(agency_id, idempotency_key)`, so different agencies can use the same key. The request fingerprint is a comparison value derived from reservation fields; it distinguishes a retry from reuse of a key with different data. No hashing algorithm is required by this design.

## Selected flow

1. The gateway validates the JWT. The service gets the authenticated `agencyId`.
2. The service looks for a non-expired idempotency record for that agency and key.
3. If it finds one with the same request fingerprint, it returns the stored response without running the reservation flow again.
4. If it finds one with a different fingerprint, it returns `409 Conflict`.
5. If no non-expired record exists, the transaction first removes any expired record for that same key, then claims the key, confirms inventory, creates the reservation when possible, and stores the final response.
6. The transaction commits. The reservation, inventory update, and idempotency record become visible together.

The unique constraint also handles two identical concurrent requests. One transaction claims the key; the other waits, then replays its result. If the first rolls back, its claim and reservation disappear, so the retry can be processed safely.

## Example: retry after a lost response

The first request succeeds, but its response is lost before it reaches the agency:

```text
Idempotency-Key: reservation-attempt-abc
Body: hotel_123, Deluxe Room, 2026-10-10 to 2026-10-12, 1 room, 2 adults

Result in PostgreSQL: reservation res_123, status CONFIRMED
```

Five seconds later, the agency retries with the same key and fields. The service returns the original result without locking availability or creating another reservation.

```http
HTTP/1.1 201 Created
Location: /v1/reservations/res_123
```

```json
{
  "reservationId": "res_123",
  "status": "CONFIRMED",
  "totalPrice": {
    "amount": "24000",
    "currency": "JPY",
    "taxesAndFeesIncluded": true
  }
}
```

## Example: same key, different request

The same key cannot change the reservation it represents.

```text
Existing key: reservation-attempt-abc
Existing request: 1 Deluxe Room
New request with the same key: 2 Deluxe Rooms
```

The service compares the request fingerprint and rejects the new request:

```http
HTTP/1.1 409 Conflict
Content-Type: application/problem+json
```

```json
{
  "type": "https://api.travel-platform.example/problems/idempotency-key-reused",
  "title": "Idempotency key belongs to a different reservation request",
  "status": 409,
  "detail": "Use a new Idempotency-Key for a new reservation request.",
  "requestId": "req_01HXYZ"
}
```

Changing an existing reservation is a separate operation with its own endpoint and rules. It is outside this first version.

## Retention and cleanup

The first version keeps records for a configurable 24-hour window. It writes `expires_at` and periodically removes expired records; a request also removes its own expired key before claiming it, so correctness does not depend on exact cleanup timing.

Within the window, the same key and fields return the original result. Afterwards, the agency uses `reservationId` to retrieve an old reservation rather than reusing the key. Cleanup never deletes reservations; retention and archiving are separate business decisions.

## Optional Redis response cache

PostgreSQL is sufficient because it is durable and part of the reservation transaction. If retry traffic becomes significant, Redis can cache a completed response after PostgreSQL commits. A cache miss or outage falls back to PostgreSQL; Redis never decides whether a reservation exists.
