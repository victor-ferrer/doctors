# 0001 — Region-partitioned topology

## Status
Accepted

## Context
The stated scale (~50,000 doctors, ~5,000,000 appointments/year, ~200 searches/sec peak) is comfortably within the capacity of a single, well-indexed relational database. Any argument for partitioning has to be honest about that rather than dressing up a residency/isolation decision as a scaling one.

The domain has a structural property that changes the calculus: appointments are inherently local (a patient in Boston books a doctor in Boston), and the brief confirms there are no cross-region transactions anywhere in the system. The only state that is genuinely global is the specialty registry, which is small, read-heavy, and explicitly tolerant of staleness.

## Options considered

**A — Single global database, single regional deployment.**
Sufficient on throughput. Rejected because: (1) it puts every patient's email + medical specialty in one jurisdiction regardless of where they live, which is a hard sell under GDPR and equivalent regimes without per-row residency machinery bolted on after the fact; (2) any outage is a global outage, contradicting the 24/7 requirement; (3) every request pays cross-continent latency for at least some users no matter where the database sits.

**B — Region-partitioned, each region authoritative for its own doctors/appointments (chosen).**
Matches the domain's natural locality. Each region is small and runs on modest infrastructure — this is explicitly not "shard for scale," it's "keep each geography's data and failure domain separate." No cross-shard transactions are needed because no use case ever spans two regions.

**C — Sharding by doctor ID hash (throughput-style sharding) instead of by geography.**
Rejected: it would achieve horizontal scale the system doesn't need, while failing to deliver residency or geography-correlated failure isolation, since a hash shard mixes doctors from every jurisdiction together.

## Decision
Region-partitioned (Option B), with regions drawn along geographic/jurisdictional lines (e.g. one region per major regulatory zone — EU, US, etc.), not by load.

## Consequences
- No cross-region transaction support is ever needed — simplifies the backend considerably.
- The specialty registry is the one deliberate exception to "everything lives in one region," and it needs its own consistency model (`0003-global-specialty-registry.md`).
- A city must be assigned to exactly one region at onboarding time; moving a city between regions later (regulatory boundary changes, for instance) is an operational migration, not something the running system needs to support dynamically.
- This is honestly a compliance/latency/isolation decision, not a scaling decision, and the submission says so rather than implying the load figures demanded it.
