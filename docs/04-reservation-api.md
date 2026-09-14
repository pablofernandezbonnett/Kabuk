# 4. Reservation API

## Goal

An agency creates a reservation after selecting one available room option. Search is not a hold, so the service checks current price and inventory again. The first version reserves one room type at one hotel for one date range, with a quantity of that type. Several room types are a later change, described in [Part 1](01-architecture.md#one-room-type-per-reservation).

## Mental model

The agency states what it wants. The service takes its identity from the JWT, calculates price from server-side data, and confirms inventory in PostgreSQL. A reservation is created only when all checks succeed together.

## API contract

```http
POST /v1/reservations
```

The request requires these headers:

```http
Authorization: Bearer <JWT>
Idempotency-Key: <client-generated-key>
Content-Type: application/json
```

The JWT identifies the agency. The request body has no `agencyId`, so an agency cannot create a reservation on behalf of another.

`Idempotency-Key` identifies one logical reservation attempt. The agency reuses it after a timeout so a retry cannot create a second reservation. Storage and concurrency behaviour are described in [Part 6](06-duplicate-requests.md).

## Request body

```json
{
  "hotelId": "hotel_123",
  "roomTypeId": "deluxe",
  "checkIn": "2026-10-10",
  "checkOut": "2026-10-12",
  "numberOfRooms": 1,
  "guestCount": 2
}
```

| Field | Description | Validation |
| --- | --- | --- |
| `hotelId` | Selected hotel. | Must exist. |
| `roomTypeId` | Selected room type. | Must exist and belong to `hotelId`. |
| `checkIn` | First stay date. | ISO 8601 date: `YYYY-MM-DD`. Cannot be in the past. |
| `checkOut` | First date after the stay. | Must be after `checkIn`. The configured 30-night maximum applies. |
| `numberOfRooms` | Number of rooms of the selected type. | Integer greater than or equal to 1. |
| `guestCount` | Total adult guests for this reservation. | Integer greater than or equal to 1 and within the selected rooms' adult capacity. |

`guestCount` means adults. Children, child ages, cots, and guest allocation to individual rooms need extra rules, so they are outside this contract. `babyCot: ON_REQUEST` can appear in search but is not requested or confirmed here.

## Price and reservation status

The client does not send a price. The service calculates it from its own availability and pricing data, so a caller cannot change it.

The returned price includes mandatory taxes and fees that the service can calculate. Optional add-ons are outside the first version. A special agency rate comes from the authenticated `agencyId` and server-side data, never caller-provided price data.

There is no payment step in this assignment. After the inventory and reservation transaction succeeds, the reservation status is `CONFIRMED`.

With future payments, a reservation could become `PENDING_PAYMENT` before `CONFIRMED`. Payment failures, expiry, and compensation rules make that a separate future flow.

## Successful response

```http
HTTP/1.1 201 Created
Location: /v1/reservations/res_123
Content-Type: application/json
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

The response returns the identifier, final status, and actual price, which may have changed since search. It does not repeat data the client sent. A later `GET /v1/reservations/{reservationId}` can return details after checking reservation ownership.

## Failure responses

| Situation | Response | Reason |
| --- | --- | --- |
| Missing `Idempotency-Key`, malformed body, or invalid dates, guests, or room count | `400 Bad Request` | The client must correct the request. |
| Missing, invalid, or expired JWT | `401 Unauthorized` | The gateway cannot authenticate the caller. |
| Valid JWT without reservation permission | `403 Forbidden` | The caller is authenticated but not allowed to reserve. |
| Unknown hotel or room type, or room type does not belong to the hotel | `404 Not Found` | The selected resource does not exist for this request. |
| Valid request but insufficient current inventory | `409 Conflict` | The request conflicts with the current inventory state. |

For example, the following response makes a missing idempotency key easy to correct:

```json
{
  "type": "https://api.travel-platform.example/problems/missing-idempotency-key",
  "title": "Missing Idempotency-Key header",
  "status": 400,
  "detail": "POST /v1/reservations requires an Idempotency-Key header.",
  "requestId": "req_01HXYZ"
}
```

`409 Conflict` means the request is valid but current inventory cannot satisfy it, for example because another reservation took the last room. The agency should search again or choose another option.

```http
HTTP/1.1 409 Conflict
Content-Type: application/problem+json
```

```json
{
  "type": "https://api.travel-platform.example/problems/insufficient-availability",
  "title": "Selected rooms are no longer available",
  "status": 409,
  "detail": "The requested number of Deluxe Rooms is no longer available for the selected dates.",
  "requestId": "req_01HXYZ"
}
```

Error responses use `application/problem+json`. They help the client act without exposing stack traces, SQL, JWT values, or full idempotency keys.
