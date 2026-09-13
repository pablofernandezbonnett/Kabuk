# 3. Large Search Results

## Goal

Osaka can have 2,500 available hotels. The API returns one small page at a time because a full response increases database work, network cost, and client processing time.

## Mental model

Pagination controls how many results the API returns at once. It does not freeze the list or reserve a room; the reservation flow still performs the final availability check.

## Pagination strategy

The first version uses offset pagination:

```http
GET /v1/hotels/availability?destinationId=dest_osaka&checkIn=2026-10-10&checkOut=2026-10-12&adults=2&rooms=1&page=1&pageSize=20&sort=PRICE_ASC
```

The backend calculates the offset as:

```text
offset = (page - 1) * pageSize
```

For example, page 3 with a page size of 20 starts after the first 40 results.

It is familiar to agencies and fits the expected 2,500 results, rather than millions.

## Page size

- Default page size: 20 hotels.
- Maximum page size: 50 hotels.
- The maximum is configuration, not a hard-coded value.
- `page` defaults to `1` and must be an integer greater than or equal to 1.
- `pageSize` must be an integer from 1 to the configured maximum.
- Invalid page values return `400 Bad Request` with a field-level validation error.

The response omits `totalCount`: an exact count adds database work and can become outdated. Instead, it uses `hasNextPage`; the backend reads one extra result. For a page size of 20, 21 results return the first 20 with `hasNextPage: true`.

Each item keeps the full hotel and room-option shape defined in [Part 2](02-availability-api.md). The shortened example below only shows the pagination fields.

```json
{
  "items": [
    {
      "hotelId": "hotel_123",
      "hotelName": "Osaka Central Hotel"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "hasNextPage": true
}
```

## Sorting

The API supports one client-selected primary sort:

| Sort value | Meaning |
| --- | --- |
| `PRICE_ASC` | Lowest total price for the requested stay first. |
| `HOTEL_STAR_RATING_DESC` | Highest official hotel category first. This is not a guest review score. |
| `DISTANCE_ASC` | Shortest distance from the platform-defined destination centre first. |

The default is `PRICE_ASC` so agencies see lower-cost options first. Every primary sort uses `hotelId ASC` as a second field, producing a deterministic order when values are equal.

For `PRICE_ASC`, the service sorts a hotel by the lowest total price among its matching room options. For `DISTANCE_ASC`, the platform must store hotel coordinates and a reference point for each destination. This is an assumption for the first version.

## Change between pages

Availability and price can change between pages, so a hotel can move, appear twice, or be missed. This does not affect booking correctness: search is not a hold, and reservations read and lock current inventory before confirmation.

## Alternative considered

Cursor pagination uses a marker from the last result. It is better for very deep pages and can reduce changed-list effects, but is harder for clients and does not support direct page navigation. It is the next option to evaluate if result sizes or traffic grow significantly.
