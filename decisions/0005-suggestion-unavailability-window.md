# 0005 — Suggestion endpoint: unavailability window, not just "free slots"

## Status
Accepted

## Context
The original use case ("a doctor requests 10 suggested alternative slots for one of her future appointments") is silent on excluding the reason she's asking. Read literally and implemented literally, "suggest alternatives" reduces to "list this doctor's other free slots" — which is broken for the motivating scenario the brief itself supplies: a doctor out sick on Tuesday, asking to relocate her Tuesday patients. Her Tuesday slots are, by every stored fact the system has (declared availability, existing bookings), still "free." A literal implementation would suggest more Tuesday appointments inside the very absence that prompted the request.

## Options considered

**A — Implement the literal requirement as written** (free slots only, ranked by displacement).
Rejected. It's not merely incomplete, it's actively wrong for the stated motivating scenario — it would suggest slots inside a known absence.

**B — Require the doctor to first record a standing "time off" entry, treated like negative availability.**
Considered, and reusable in principle, but rejected as the *only* mechanism: it presumes the doctor plans her absence in advance and maintains it as durable schedule state, when the motivating case (calling in sick) is exactly the situation where there's no time to do that, and no reason the exception should become a permanent fixture of her `availability_rule` data.

**C — Accept an ad-hoc, request-scoped unavailability window as a parameter to the suggestion endpoint itself (chosen).**
The doctor supplies the window she needs avoided (e.g. "all of Tuesday") as part of the same request asking for alternatives. It requires no standing schedule change, applies only to this suggestion computation, and directly encodes the actual constraint driving the request.

## Decision
Option C: the suggestion endpoint's contract includes a doctor-supplied unavailability window, filtered out of the candidate slot list *in addition to* declared availability and existing bookings (`04-suggestion-algorithm.md` §3, step 2).

## Consequences
- The endpoint's contract is one parameter richer than the original use-case list implies, and that's called out explicitly here rather than silently shipped as if it were always obvious.
- The unavailability window is request-scoped, not persisted — it doesn't touch `availability_rule` or require a "time off" domain concept to exist. If a future requirement wants standing, plannable absences (a real vacation calendar), that's a larger feature this ADR deliberately does not attempt to solve; this fix addresses exactly the case in scope.
- The window's absence in a request degrades gracefully to the literal-requirement behavior (plain free-slot suggestions) — the parameter is additive and optional, so this decision doesn't force every caller of the endpoint to always supply one.
