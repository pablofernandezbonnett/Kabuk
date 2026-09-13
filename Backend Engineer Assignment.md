# Mid-Level Backend Engineer — Technical Assignment

**Expected effort:** 3–4 hours maximum
**Deadline:** 4 days from receipt
**Role:** Backend Engineer, Platform

## Purpose

Evaluate practical backend engineering ability, technical reasoning, and readiness to independently own bounded production services.

## Candidate Instructions

This assignment is intentionally open-ended.

We are not looking for one specific architecture or technology choice.

We are evaluating:

* How you understand requirements
* How you design a backend service
* How you structure an API
* How you model and persist data
* How you think about concurrency and correctness
* How you handle failures
* How you test your implementation
* How you explain technical decisions and trade-offs

You may make reasonable assumptions.

Please document those assumptions.

You **do not need to build a complete production application**.

We strongly recommend limiting your work to **3–4 hours**.

A concise and well-reasoned solution is preferred over a large or overly complicated submission.

---

# Scenario

You are building a new backend service for a travel platform.

The platform allows travel agencies to search for hotels and create reservations.

For this assignment, assume the platform currently contains:

* 100,000 hotels
* Multiple room types per hotel
* Availability by date
* Pricing by room and date
* Travel agencies accessing the platform through APIs

You are responsible for designing a small **Hotel Availability & Reservation Service**.

---

# Part 1 — Architecture

Create a simple architecture diagram showing how you would structure the service.

At minimum, consider:

* External API
* Backend application/service
* Database
* Availability data
* Reservation data

You may introduce additional components if you believe they are useful.

Examples could include:

* Cache
* Queue
* Background worker
* API gateway
* Monitoring

However, additional components should only be introduced where they solve a specific problem.

## Explain

For each major component:

* What is its responsibility?
* Why does it exist?
* How does it communicate with the other components?

---

# Part 2 — Hotel Availability API

Design an API that allows a travel agency to search for available hotels.

A request should support at least:

* Destination
* Check-in date
* Check-out date
* Number of adults
* Number of rooms

You may include additional parameters if appropriate.

## Provide

1. HTTP method
2. Endpoint
3. Example request
4. Example successful response
5. Example validation/error response

## Example Scenario

A travel agency wants:

* **Destination:** Osaka
* **Check-in:** October 10
* **Check-out:** October 12
* **Adults:** 2
* **Rooms:** 1

Your API should return hotels and rooms that can be booked for those dates.

---

# Part 3 — Large Search Results

Assume Osaka has:

**2,500 available hotels**

The client should not receive all 2,500 results in a single response.

Explain how you would handle this.

Include:

* Pagination strategy
* Page/result size
* Any sorting considerations

You may use:

* Offset pagination
* Cursor pagination
* Another approach

Explain why you selected your approach.

---

# Part 4 — Reservation API

Design an API that creates a hotel reservation.

## Provide

1. HTTP method
2. Endpoint
3. Example request
4. Example successful response
5. Example failure response

A reservation should contain at least:

* Hotel
* Room type
* Check-in
* Check-out
* Number of guests
* Travel agency
* Reservation status

---

# Part 5 — Concurrency Scenario

A hotel has exactly:

**1 room remaining**

Two customers attempt to book the room at almost exactly the same time.

Both requests reach your backend.

## Question

How would you guarantee that only one reservation succeeds?

Please explain:

* What happens in the application
* What happens in the database
* How the second request is handled

You may include pseudocode, SQL, or a sequence diagram if useful.

---

# Part 6 — Duplicate Request

## Scenario

A travel agency sends a reservation request.

Your service successfully creates the reservation.

However, the travel agency experiences a network timeout and does not receive the response.

Five seconds later, they send the same request again.

## Question

How would you prevent two reservations from being created?

Explain your approach.

---

# Part 7 — Database Design

Design a simplified database model supporting the system.

At minimum, consider:

* Hotels
* Room Types
* Availability / Inventory
* Reservations

Provide either:

* ER diagram

or

* Table definitions

You do not need to include every possible field.

## Explain

* Primary keys
* Important foreign keys
* Important indexes
* Any important constraints

---

# Part 8 — Performance Scenario

The availability API normally responds in:

**250 ms**

One day it begins taking:

**4–5 seconds**

Investigation suggests that the database query is slow.

## Question

How would you investigate the issue?

Describe the steps you would take.

You do not need to provide an exact solution because the root cause is intentionally unknown.

---

# Part 9 — Testing Strategy

Explain how you would test this service.

Include examples of what you would cover with:

## Unit Tests

What business logic should be tested?

## Integration Tests

What components should be tested together?

## API Tests

What API behavior should be tested?

You do not need to provide complete test code.

One or two example test cases are sufficient.

---

# Part 10 — Production Readiness

Assume this service is going to production.

What would you want to monitor?

Consider areas such as:

* API performance
* Errors
* Database
* Reservations
* Infrastructure

You do not need to design a complete observability platform.

List the signals you believe would be most important.

---

# Part 11 — Trade-Offs

Briefly answer:

* What part of your design would you improve if you had another week?
* What part of your design is intentionally simple?
* What assumptions did you make?
* What do you believe is the largest technical risk in your design?

---

# Optional — Implementation

Implementation is **optional**.

If you would like, you may implement one of the following:

## Option A

The hotel availability endpoint.

## Option B

The reservation endpoint.

## Option C

The concurrency protection logic for reservations.

You may use any language or framework you are comfortable with.

Examples:

* C# / .NET
* Java / Spring
* Go
* Python
* Node.js / TypeScript

Do not spend more than the recommended assignment time implementing code.

Architecture and reasoning are more important than the amount of code produced.

---

# Submission

Please provide:

1. Architecture diagram
2. API design
3. Database design
4. Answers to the scenarios
5. Testing approach
6. Production-readiness considerations
7. Trade-offs and assumptions
8. Optional implementation, if completed

Acceptable formats include:

* Markdown
* PDF
* Notion page
* GitHub repository

---

# AI Tool Usage

AI development tools may be used.

Examples include:

* ChatGPT
* Claude
* Cursor
* GitHub Copilot
* Codex

If AI tools are used, please briefly describe:

* Which tools you used
* What you used them for

You remain responsible for understanding and defending everything contained in your submission.

During the follow-up interview, you may be asked to explain or modify any portion of your design.
