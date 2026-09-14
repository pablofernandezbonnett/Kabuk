# 8. Performance Investigation

## Goal

The availability API normally responds in 250 ms but now takes 4–5 seconds. The exercise suggests a slow database query, but I would confirm the evidence before changing the query, adding an index, or increasing capacity.

## Mental model

API time can come from application work, waiting for a database connection, database work, or the network. Monitoring and logs identify the slow part; the execution plan explains the query.

## Investigation steps

### 1. Confirm where time is spent

Use monitoring, tracing or a profiler, and logs to connect the slow request to its database call. Check:

- API latency and database-query duration;
- request rate, error rate, and database connection-pool usage;
- database CPU, memory, disk I/O, and connection count; and
- the query identifier and safe example input that reproduces the problem.

This confirms whether database work, rather than application code or the network, caused the extra time.

### 2. Identify a representative slow query

Find the actual availability `SELECT` in slow-query logs, tracing, or database query statistics. Record representative parameters such as destination, dates, guest count, page, and sort order. A long stay or popular destination may expose a different problem from a normal search.

### 3. Reproduce safely before testing production

Use pre-production with anonymised, production-like data and realistic traffic where possible. It lets us inspect and load-test the query without affecting agencies or reservations.

If it cannot be reproduced there, observe production in a tightly controlled way only when necessary. Never run an uncontrolled load test in production. A load test in pre-production can reproduce the traffic pattern and compare a proposed change.

### 4. Inspect the execution plan

For the identified availability `SELECT`, run:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

- `EXPLAIN` shows the planned steps; `ANALYZE` runs the query and adds actual row counts and timing.
- `BUFFERS` shows whether the work used cached data, disk reads, or temporary storage.

`ANALYZE` executes the statement; it does not only describe it. It is appropriate for this controlled read-only availability query. I would not run it carelessly on `UPDATE`, `DELETE`, or reservation statements because they make real changes.

### 5. Form a hypothesis from the plan and database state

I would look for the part using the time:

- **Too many rows read:** a filter may not reduce the data enough.
- **Wrong row estimate:** planned and actual row counts differ greatly. This can suggest stale statistics or unusual data.
- **Expensive join, sort, or aggregation:** these operations may process too many rows or use temporary disk space.
- **Disk I/O:** reading much data from disk is slower than reading cached data.
- **Waiting:** PostgreSQL sessions can show a lock or I/O wait. A normal availability read should not wait long on reservation locks.

A sequential scan is not always a problem. When a query needs a large part of a table, it can be cheaper than repeatedly using an index.

### 6. Test the smallest targeted change

The evidence decides the next step: refresh statistics, correct the query, or add one index for a frequent and selective filter or sort. A column does not need an index only because it is in `WHERE` or is a foreign key.

Test one change in pre-production with the same query parameters and traffic. Compare response time, rows processed, buffer activity, and the effect on writes. Keep it only when it solves the measured problem without an unacceptable write cost.

### 7. Release and verify

Deploy the validated change with normal safeguards. Keep watching API latency, query time, database load, errors, and connection use. If it does not help under real traffic, roll it back and continue from the new evidence.
