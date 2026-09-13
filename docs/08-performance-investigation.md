# 8. Performance Investigation

## Goal

The availability API normally responds in 250 ms but now takes 4–5 seconds. Although the exercise suggests a slow database query, the goal is evidence before changing the query, adding an index, or increasing capacity.

## Mental model

API time includes application work, connection waits, database work, and network time. Monitoring and logs locate the slow part; the execution plan explains the identified query.

## Weak approach vs better approach

**Weak approach:** add an index because the query is slow.

**Better approach:** confirm the slow query, inspect its real execution, form a small hypothesis, test one change safely, and compare the result with the same workload. An index can help a selective filter or sort, but it also makes writes more expensive and is not always the right fix.

## Investigation steps

### 1. Confirm where time is spent

Use monitoring, tracing or a profiler, and logs to correlate the slow request with its database call. Check:

- API latency and database-query duration;
- request rate, error rate, and database connection-pool usage;
- database CPU, memory, disk I/O, and connection count; and
- the query identifier or parameterised query, with the input values that reproduce the problem.

The purpose is to confirm that database work, rather than application code or the network, accounts for the extra time.

### 2. Identify a representative slow query

Find the actual availability `SELECT` from slow-query logs, tracing, or database query statistics. Record representative search parameters, for example destination, dates, guest count, page, and sort order. A query that is slow only for a long stay or a popular destination may have a different cause from the usual search.

### 3. Reproduce safely before testing production

The preferred environment is pre-production with anonymised, production-like data and a realistic amount of traffic. This makes it possible to inspect and load-test the query without affecting agencies or reservations.

If it cannot be reproduced there, use tightly controlled production observation only when necessary. Do not run an uncontrolled load test against production. A load-testing tool in pre-production can reproduce the traffic pattern and compare a proposed change.

### 4. Inspect the execution plan

For the identified availability `SELECT`, run:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

- `EXPLAIN` shows the expected plan; `ANALYZE` runs the query and adds actual row counts and timing.
- `BUFFERS` shows work using cached data, disk reads, or temporary storage.

`ANALYZE` executes the statement; it is not only an explanation. It is appropriate for this controlled read-only availability query. I would not run it carelessly on `UPDATE`, `DELETE`, or reservation statements, because those statements would make real changes.

### 5. Form a hypothesis from the plan and database state

I would look for the part that consumes the time, rather than assuming the first scan is wrong:

- **Too many rows read:** a filter may not narrow the data as expected.
- **Wrong row estimate:** the estimated rows and actual rows differ greatly, which can suggest stale statistics or unusual data distribution.
- **Expensive join, sort, or aggregation:** joining, ordering, or grouping may use more rows or temporary disk space than expected.
- **I/O pressure:** many reads from disk can be slower than data already in memory.
- **Waiting:** active PostgreSQL sessions can expose a lock or I/O wait. A normal availability read should not wait on reservation locks for a long time, so this would be a useful clue.

A sequential scan is not automatically a problem. If a query needs a large part of a table, reading it sequentially can be cheaper than repeatedly using an index.

### 6. Test the smallest targeted change

The next action depends on the evidence: refresh statistics, correct the query, or add one index for a frequent and selective filter or ordering pattern. An index is not justified only because a column is in `WHERE` or is a foreign key.

Test one change in pre-production using the same query parameters and traffic profile. Compare response time, rows processed, buffer activity, and the effect on write operations. Keep the change only when it improves the measured problem without creating an unacceptable write cost.

### 7. Release and verify

Deploy the validated change with normal safeguards and continue watching API latency, query time, database load, errors, and connection usage. If it does not hold under real traffic, roll back and continue with the new evidence.
