# 2. Hotel Availability API

## Goal

Travel agencies need room types that can satisfy a destination, date range, number of adults, and number of rooms.

## Mental model

The availability API answers: "What can this agency book now?" It does not hold a room. The reservation flow checks current inventory and price again before confirmation.

## API contract

```http
GET /v1/hotels/availability
```

Query parameters fit this small, read-only set of filters.

| Parameter | Required | Description | Validation |
| --- | --- | --- | --- |
| `destinationId` | Yes | Stable ID from the platform destination catalogue. | Must use a valid identifier format. |
| `checkIn` | Yes | First stay date. | ISO 8601 date: `YYYY-MM-DD`. Cannot be in the past. |
| `checkOut` | Yes | First date after the stay. | ISO 8601 date: `YYYY-MM-DD`. Must be after `checkIn`. |
| `adults` | Yes | Total number of adult guests. | Integer greater than or equal to 1. |
| `rooms` | Yes | Number of rooms of the same room type. | Integer greater than or equal to 1. |

Dates are calendar dates, not timestamps. Java uses `LocalDate` to avoid ambiguous formats such as `10/12/2026` and an unnecessary UTC time.

The maximum stay is 30 nights, configured rather than hard-coded so it can change without changing the API contract.

## Example request

```http
GET /v1/hotels/availability?destinationId=dest_osaka&checkIn=2026-10-10&checkOut=2026-10-12&adults=2&rooms=1
Authorization: Bearer <JWT>
```

`dest_osaka` is an example identifier. The client can show "Osaka", while the API receives the stable ID.

## Search result rules

The response contains room options that satisfy the request. One selected option represents one room type: `availableRooms` must meet `rooms`, and `maxAdultsPerRoom * rooms` must meet `adults`. A later hotel-details endpoint can return a full amenity list without making search results too large.

## Example successful response

```json
{
  "items": [
    {
      "hotelId": "hotel_123",
      "hotelName": "Osaka Central Hotel",
      "destination": {
        "id": "dest_osaka",
        "name": "Osaka"
      },
      "hotelHighlights": ["Free WiFi", "Swimming pool"],
      "roomOptions": [
        {
          "roomTypeId": "deluxe",
          "roomTypeName": "Deluxe Room",
          "bedDescription": "1 extra-large double bed",
          "maxAdultsPerRoom": 2,
          "availableRooms": 3,
          "roomFeatures": ["Hairdryer"],
          "offer": {
            "breakfastIncluded": true,
            "babyCot": "ON_REQUEST"
          },
          "totalPrice": {
            "amount": "24000",
            "currency": "JPY",
            "taxesAndFeesIncluded": true
          }
        }
      ]
    }
  ]
}
```

`availableRooms` helps an agency choose, but is not a booking guarantee. `totalPrice` is the final price for the requested rooms and dates, including all known mandatory taxes and fees. `babyCot: ON_REQUEST` requires property confirmation.

The first version returns one default offer per room type. Several rates later can add an `offerId` or `ratePlanId`.

## Validation and errors

The API returns `400 Bad Request` for malformed or invalid input. Its `application/problem+json` response includes field details that a client can correct.

```http
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json
```

```json
{
  "type": "https://api.travel-platform.example/problems/validation-error",
  "title": "Invalid availability search request",
  "status": 400,
  "detail": "One or more request fields are invalid.",
  "errors": [
    {
      "field": "checkOut",
      "code": "AFTER_CHECK_IN_REQUIRED",
      "message": "checkOut must be after checkIn."
    },
    {
      "field": "rooms",
      "code": "MINIMUM_VALUE",
      "message": "rooms must be at least 1."
    }
  ],
  "requestId": "req_01HXYZ"
}
```

| Situation | Response | Reason |
| --- | --- | --- |
| Invalid parameter format or invalid dates, adults, rooms, or stay length | `400 Bad Request` | The client must correct the request. |
| Valid destination ID format but unknown destination | `404 Not Found` | The client refers to a destination that does not exist in the catalogue. |
| Valid search with no available hotels | `200 OK` and `items: []` | The search was valid but found no matching inventory. |

## Alternative considered

A future `POST /v1/hotel-searches` endpoint could accept complex filters, such as different guest counts per room, child ages, and preferences.

```json
{
  "destinationId": "dest_osaka",
  "checkIn": "2026-10-10",
  "checkOut": "2026-10-12",
  "rooms": [
    { "adults": 2 },
    { "adults": 1 }
  ],
  "preferences": ["SWIMMING_POOL", "BREAKFAST_INCLUDED"]
}
```

This is outside the first version because the required filters are simple; `GET` is easier to read, test, and use.

Pagination, page size, and sorting are described separately in [Part 3](03-large-search-results.md).
