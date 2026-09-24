# 0002 — Slots computed on read, not materialized

## Status
Accepted

## Context
A doctor's free slots could either be (a) materialized ahead of time as rows in a `slot` table whenever her availability changes, or (b) computed on demand from her availability rules minus her existing appointments. The brief fixes (b) and asks the submission to explain how that meets the p99 < 300ms search target.

## Options considered

**A — Materialized slot table**, one row per (doctor, potential start instant), regenerated whenever availability rules change, and flipped to "held" on booking.
Rejected. It reintroduces exactly the problem §7 is designed to avoid: an availability edit either has to regenerate a horizon's worth of future slot rows (up to 90 days × doctor's daily slot count) or leave stale ones, and reconciling "which materialized slots correspond to already-booked appointments after an availability edit" is precisely the "availability changes never void existing appointments" problem, except now expressed as a data-migration problem on every edit instead of a read-time computation. It also duplicates the DST decision in two places (materialization time and appointment time) instead of one.

**B — Computed on read (chosen).**
A slot is `availability_rule ⊕ booking_horizon ⊖ existing_appointments`, evaluated at request time, per doctor. No slot ever needs to be regenerated because none is ever stored. An availability edit takes effect on the very next read, atomically, with no migration step and no window where stored slots disagree with the rules that generated them.

## Decision
Computed on read (Option B), made tractable by two constraints that keep the "on read" cost bounded and cache-friendly:

1. **Search is paginated over doctors (20 per page), never over individual slots** (§9 of the brief) — so a single search request computes slots for a bounded, small number of doctors, not for a whole city.
2. **Slot computation is deterministic and cacheable**, keyed on `(doctor_id, rules_version, date/month)` — see `02-data-model.md` §3. A cache hit costs a lookup; a cache miss costs one bounded computation over a handful of availability rules and a handful of existing appointments, both served by indexes.

## Consequences
- No background job ever regenerates slot data — one entire class of consistency bug (stale materialized slots) doesn't exist.
- The tradeoff is CPU-per-search-request instead of storage-per-doctor; at this scale, and with the cache in front of it, that tradeoff favors read simplicity over write simplicity, which matches "strongly read-heavy" from §2.
- Correctness of the p99 target now rests on cache hit rate and on the two supporting indexes (`idx_search`, `idx_doctor_local_date`) rather than on index-only lookups against a precomputed table — this is made explicit in `02-data-model.md` rather than left as an assumption.
