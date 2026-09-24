# 0004 — Transactional outbox for notification dispatch

## Status
Accepted

## Context
Token-based patient cancellation, doctor-initiated cancellation, and rescheduling all require outbound email to function (§10 of the brief). Email delivery must never be able to roll back a confirmed, valid appointment — an SMTP timeout is not a reason to un-book a patient.

## Options considered

**A — Synchronous send inside the booking transaction, or as a blocking call right after it.**
Rejected outright per §10: couples booking correctness to an external system's availability and latency, and makes an appointment's fate depend on whether an email provider is having a bad day.

**B — Message broker / CDC pipeline (Kafka + Debezium reading the DB's write-ahead log).**
Technically solid and at-least-once by construction, but it's infrastructure sized for a scale and team topology this system doesn't have: a modest, single-region-deployed regional API with a small ops footprint. Standing up and operating a broker per region to deliver a comparatively low volume of transactional emails is disproportionate to the problem.

**C — Transactional outbox table + in-process polling dispatcher (chosen).**
The state-changing use case writes the domain row and an outbox row in the same database transaction. A separate `@Scheduled` job polls `notification_outbox WHERE dispatched_at IS NULL`, using `FOR UPDATE SKIP LOCKED` so multiple stateless API instances can run the poller without double-sending, and calls the email provider outside any request/response cycle.

## Decision
Option C.

## Consequences
- At-least-once delivery: a crash between "email sent" and "row marked dispatched" produces a duplicate email, which is an acceptable failure mode for a medical appointment notification (never silently losing one matters far more than never duplicating one).
- Booking latency is bounded by the database write alone; email latency/availability never appears on that path.
- The dispatcher is a small in-process component, not a new piece of infrastructure to operate — consistent with each region running on "modest infrastructure" (§2).
- If email volume or delivery-guarantee requirements grow materially beyond what this brief describes, the outbox table is the natural seam to swap in a broker later (a CDC connector can read the same table) without changing how use-cases write to it.
