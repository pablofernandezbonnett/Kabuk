# 5. Concurrency

## Why this matters

Availability search is only a snapshot. Two agencies can both see one room left and then try to reserve it at nearly the same time. The service must confirm at most one reservation.

## Mental model

The application does not decide who receives the last room. PostgreSQL does.

The daily availability row is the source of truth. The service asks PostgreSQL to lock that row before it reads and changes it. This works even when the service runs on several application instances.

### Why row locks are needed

Without a lock, two requests can both read `availableRooms = 1` before either updates it. Both might then create a reservation.

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

A short transaction starts immediately before locking inventory and ends as soon as the database changes commit or roll back. It only locks and checks inventory, reduces it, creates the reservation, and stores the idempotency result. It makes no payment, email, or other slow external call, so competing requests do not wait longer than necessary.

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

A and B are only labels. PostgreSQL's database lock manager lets whichever transaction obtains the lock first continue; the other waits. The application does not choose a winner.

If request A fails before commit, PostgreSQL rolls back its inventory change and releases the lock. Request B can then lock the row and may succeed.

## Alternative considered: conditional update

A conditional update reduces inventory only where `available_rooms >= numberOfRooms`. It is compact for one date, but a multi-night stay still needs a transaction that checks every date and rolls back if one fails. We chose row locks because the sequence is clearer to explain.

## Failure boundary

`409 Conflict` means the inventory check completed and there were not enough rooms. An unexpected database error or transaction timeout is not an availability result; it is a server failure and should be logged and handled as a `5xx` response.
