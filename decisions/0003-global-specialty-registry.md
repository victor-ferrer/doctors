# 0003 — Global specialty registry with regional read replicas

## Status
Accepted

## Context
§4 of the brief states a genuine tension rather than hiding it: a canonical, curated registry implies curation latency; the requirement that a country's specialties are usable immediately, before any curator has looked at them, implies the opposite. These pull against each other and the design has to resolve the pull, not paper over it.

Separately, §2 establishes that the specialty registry is the one piece of state shared across an otherwise region-partitioned system, and that it's read-heavy and stale-tolerant.

## Options considered

**A — Fully synchronous global registry**, every region reads and writes it directly on every request.
Rejected. This reintroduces a single global dependency into an architecture whose entire point (ADR 0001) is that regions don't depend on each other at request time. A global outage would then degrade specialty search everywhere, contradicting the isolation goal.

**B — Fully regional registries, no sharing at all.**
Rejected. It would mean re-curating the same canonical mapping independently per region for specialties that exist in multiple countries a region might serve, and would make the curated canonical concept meaningless as a cross-country concept — the registry's whole purpose (§4: seed from SNOMED CT / HL7, but stay standard-agnostic) is to have one canonical concept per specialty, not N regional guesses at it.

**C — One global write path, async pull-replicated regional read copies (chosen).**
The canonical registry and its country-localized labels/aliases are owned by a small global service with its own database. Each region periodically pulls deltas (by `updated_at`) into a local read-only cached copy (`country_specialty`, `country_specialty_alias` tables inside the regional DB, per `02-data-model.md`). Regional reads — every doctor-profile write and every patient search — hit the local copy, never the global service.

## Decision
Option C, and it directly resolves the curation-vs-immediacy tension:

- **A country adds a locally-valid specialty immediately** by writing to the global registry with `canonical_specialty_id = NULL`. This write does need the global service to be reachable, but it is a rare, admin-rate operation (onboarding), not a request-rate one — unlike search and booking, it is allowed to have a brief dependency on the global tier being up.
- **That new row is usable in doctor profiles and patient search from the moment the region's next sync pulls it** — typically within minutes, not blocked on any curator.
- **A curator later sets `canonical_specialty_id`**, an update that is also picked up on the next regional sync. Nothing that already referenced the row (doctor profiles, past search results) needs to change — the row's identity never changes, only whether it's mapped.

## Consequences
- Regions remain autonomous for the read path (search, doctor profiles) even during a global-tier outage, using their last-synced copy — consistent with the 24/7 requirement.
- Onboarding a new country's first specialty has a brief dependency on the global service; this is called out explicitly rather than silently assumed away, since honesty about this edge is exactly what the brief's "state the tension" instruction is asking for.
- "Tolerates staleness" is a real, bounded property here: the sync interval (minutes) is a tuning knob, not an open-ended risk, because the registry's write rate (new specialties, curator edits) is very low next to its read rate.
- An unmapped specialty is fully functional for search and booking; the canonical mapping only matters for cross-country reporting/analytics use cases that are out of scope for this brief.
