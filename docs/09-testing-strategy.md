# 9. Testing Strategy

## Goal

The service must return correct searches, prevent overselling, and keep retries safe. Each test type checks a different boundary: unit tests cannot prove PostgreSQL locking, and database tests cannot prove the public HTTP contract.

## Mental model

Test the rule where it lives:

- **Unit tests** test one business rule without a database or HTTP server.
- **Integration tests** test our service, transaction handling, and a real PostgreSQL database together.
- **API tests** call the service through HTTP and check the contract seen by an agency.

Most tests are unit tests because they are fast. Fewer integration and API tests cover the high-risk paths that unit tests cannot prove. This service has no browser UI, so browser tests are outside this scope.

## Why use Gherkin-style scenarios

`Given`, `When`, and `Then` make the initial data, action, and outcome clear to technical and non-technical readers. They document expected behaviour and can later become Java tests.

## Unit tests

Unit tests cover local rules. They do not prove inventory or locking; those behaviours belong to integration tests.

Examples of unit-test coverage:

- stay dates are valid and do not exceed the configured maximum of 30 nights;
- the requested adult count fits the selected number of rooms and their capacity;
- the server calculates the final price from daily availability data, rather than accepting a caller-provided price; and
- the same reservation body produces the same idempotency request fingerprint.

### Example: server-side price calculation

```gherkin
Scenario: Calculate the total price for a two-night stay
  Given one Deluxe Room costs 12000 JPY on 10 October
  And one Deluxe Room costs 12000 JPY on 11 October
  And the known mandatory taxes and fees are included in those amounts
  When an agency requests one room from 10 October to 12 October
  Then the service calculates a total price of 24000 JPY
  And the result marks taxesAndFeesIncluded as true
```

This proves the price rule without HTTP or a database.

## Integration tests

Integration tests run the reservation service, transaction handling, constraints, and PostgreSQL row locks together. They use an isolated PostgreSQL instance in Docker or a test container, with the same migrations and fixture data. An in-memory database or mock cannot prove SQL, constraints, transactions, or concurrency.

Examples of integration-test coverage:

- daily inventory cannot become negative;
- a room type must belong to the selected hotel;
- an idempotency record and its reservation are stored atomically; and
- a retry with the same agency, idempotency key, and body does not create another reservation.

### Example: concurrent reservation for the last room

```gherkin
Scenario: Confirm only one reservation when one room remains
  Given one Deluxe Room is available on each requested stay date
  And two different agencies request that same room and date range at the same time
  When both reservation transactions run against PostgreSQL
  Then one reservation is confirmed
  And the other request receives an insufficient-availability result
  And availability is zero, never negative
  And exactly one reservation is stored
```

This verifies the booking rule with the real database behaviour from [Part 5](05-concurrency.md).

## API tests

API tests verify the public HTTP contract: route, parameters, JSON, status, headers, and errors. They are automated HTTP requests, not manual `curl` checks or load tests. Load testing belongs to [Part 8](08-performance-investigation.md).

The external authentication service is outside this scope. API tests use a controlled identity at the gateway/API boundary and cover missing or invalid authentication as `401` and a missing scope as `403`.

### Example: availability search contract

```gherkin
Scenario: Search Osaka availability with valid input
  Given Osaka has an available Deluxe Room for the requested dates
  When an authenticated agency searches for one room for two adults
  Then the API responds with 200 OK
  And the response includes the hotel, availableRooms, offer, and totalPrice
  And totalPrice marks taxesAndFeesIncluded as true
```

The HTTP example shows the request and key response fields:

```http
GET /v1/hotels/availability?destinationId=dest_osaka&checkIn=2026-10-10&checkOut=2026-10-12&adults=2&rooms=1
Authorization: Bearer <test-JWT>
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "items": [
    {
      "hotelId": "hotel_123",
      "hotelName": "Osaka Central Hotel",
      "roomOptions": [
        {
          "roomTypeId": "deluxe",
          "availableRooms": 3,
          "offer": { "breakfastIncluded": true },
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

The same suite checks invalid input: equal `checkIn` and `checkOut` returns `400 Bad Request`, `application/problem+json`, and a `checkOut` error. A valid search with no rooms returns `200 OK` and `items: []`.
