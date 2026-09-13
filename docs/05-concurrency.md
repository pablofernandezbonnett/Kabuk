# 5. Concurrency

## Why this matters

Availability search is only a snapshot. Two agencies can both see one room left and then try to reserve it at nearly the same time. The service must confirm at most one reservation.

## Mental model

The application does not decide who receives the last room. PostgreSQL does.

The daily availability row is the source of truth. The service asks PostgreSQL to lock that row before it reads and changes it. This works even when the service runs on several application instances.

### Weak approach versus the selected approach

A weak approach is: read `availableRooms = 1`, decide that it is enough, and update later without a lock. Two requests can both read `1` before either update happens. Both might then create a reservation.

The selected approach locks the affected availability rows inside one short database transaction. Only one transaction can change a locked row at a time.

## Selected approach: row-level locks

The reservation service starts a transaction and locks the daily availability rows for the selected room type and stay dates. It locks dates in ascending order. A consistent order matters when several date rows are involved because it reduces the chance of two transactions waiting for each other.

```sql
SELECT stay_date, available_rooms
FROM room_availability
WHERE room_type_id = :roomTypeId
  AND stay_date >= :checkIn
  AND stay_date < :checkOut
ORDER BY stay_date
FOR UPDATE;
```

`FOR UPDATE` is a row-level lock. It prevents another reservation transaction from changing or locking the same availability rows until the current transaction commits or rolls back.

### What "short transaction" means

A short transaction starts immediately before the inventory rows are locked and ends as soon as the required database changes commit or roll back. It is not a fixed time limit. It contains only the database work needed for the reservation: lock rows, validate inventory, reduce it, create the reservation, and store the technical idempotency result when needed.

It must not call a payment provider, send an email, or wait for another slow external service. Keeping this critical section small reduces how long competing requests wait for locks and lowers the risk of timeouts.

The transaction then follows this sequence:

1. Lock every required daily availability row and verify that PostgreSQL returned one row for every requested stay date.
2. Verify that every date has at least `numberOfRooms` remaining.
3. Reduce the available rooms for every date.
4. Create the reservation with status `CONFIRMED`.
5. Commit all changes together.

If a date row is missing or any date does not have enough rooms, the transaction makes no inventory or reservation changes and the service returns `409 Conflict`. When an idempotency key is present, it may still commit the separate technical replay record described in [Part 6](06-duplicate-requests.md).

## Two requests for the last room

Assume exactly one room remains for the selected room type and date range.

| Step | Request A | Request B |
| --- | --- | --- |
| 1 | Starts a transaction and locks the availability row. | Starts a transaction and tries to lock the same row. |
| 2 | Reads `available_rooms = 1`. | Waits; it cannot use the row while A holds the lock. |
| 3 | Reduces availability to `0`, creates the reservation, and commits. | Continues after A commits. |
| 4 | Receives `201 Created`. | Reads the updated value `0`, rolls back, and receives `409 Conflict`. |

If request A fails before commit, PostgreSQL rolls back its inventory change and releases the lock. Request B can then lock the row and may succeed.

## Alternative considered: conditional update

Another approach is an atomic conditional update, for example: reduce availability only where `available_rooms >= numberOfRooms`.

For one date, this can be compact. For a multi-night stay, the service must still use a transaction, verify that every required date was updated, and roll back if even one date could not be reduced. It is correct when implemented carefully, but is less direct to explain than locking all stay dates, checking them, and creating the reservation in one transaction.

## Failure boundary

`409 Conflict` means the inventory check completed and there were not enough rooms. An unexpected database error or transaction timeout is not an availability result; it is a server failure and should be logged and handled as a `5xx` response.
