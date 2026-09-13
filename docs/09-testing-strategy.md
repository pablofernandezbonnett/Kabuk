# 9. Testing Strategy

## Goal

The service must return correct search results, prevent overselling, and keep retries safe. Different tests give evidence at different boundaries. A fast unit test cannot prove that PostgreSQL locks rows correctly, and a database test cannot by itself prove that the public HTTP contract is clear.

## Mental model

Test the rule where it lives:

- **Unit tests** test one business rule without a database or HTTP server.
- **Integration tests** test our service, transaction handling, and a real PostgreSQL database together.
- **API tests** call the service through HTTP and check the contract seen by an agency.

Most tests should be unit tests because they are fast and isolate business rules. Fewer integration and API tests cover the high-risk paths that units cannot prove. This service has no browser UI, so browser tests are outside the assignment scope.

## Why use Gherkin-style scenarios

The examples use `Given`, `When`, and `Then` to state initial data, action, and outcome clearly for technical and non-technical reviewers. This documents behaviour; the scenarios can later become Java tests in a suitable framework.

## Unit tests

Unit tests cover pure local rules. They do not use mocks as evidence that inventory or locking works; those behaviours belong to integration tests.

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

Integration tests verify the reservation service, transaction manager, persistence mapping, constraints, and PostgreSQL row locks together. They use a real isolated PostgreSQL instance in Docker or a test container, with the same migrations and explicit fixture data. This is more useful than an in-memory database or mock for SQL, constraints, transactions, and concurrency.

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

This test has high value because it verifies the booking invariant with the actual database behaviour described in [Part 5](05-concurrency.md).

## API tests

API tests verify the public HTTP contract: route, parameters, JSON, status, headers, and error format. They are automated HTTP requests, not manual `curl` checks or load tests; load testing belongs to [Part 8](08-performance-investigation.md).

The external authentication service is outside this assignment. API tests use a controlled authenticated identity at the gateway/API boundary and cover missing or invalid authentication as `401` and a missing scope as `403`.

### Example: availability search contract

```gherkin
Scenario: Search Osaka availability with valid input
  Given Osaka has an available Deluxe Room for the requested dates
  When an authenticated agency searches for one room for two adults
  Then the API responds with 200 OK
  And the response includes the hotel, availableRooms, offer, and totalPrice
  And totalPrice marks taxesAndFeesIncluded as true
```

The request and the important response fields are visible in the HTTP example below:

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
