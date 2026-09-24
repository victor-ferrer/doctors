# Worldwide Doctor Appointment System — Design Submission

A 24/7, region-partitioned appointment system for doctors and patients, designed against a Java backend / JavaScript frontend split, computed-on-read slot derivation, and explicit DST handling. This repository is design documents, not code (per the brief's own scoping instruction). The full source brief is at [`requirements.md`](./requirements.md).

The system deliberately partitions by geography — not for throughput (the stated load fits comfortably on one database) but for data residency, regional failure isolation, and latency — while keeping a single small global service for the one thing that's genuinely shared across regions: the specialty registry. Two independent JavaScript frontends (an anonymous patient app, an authenticated doctor portal) talk to a regional Java/Spring Boot API over plain REST/JSON, chosen specifically so frontend engineers never need to write or generate code from Java.

## Significant decisions

| Decision | Why not the alternative |
|---|---|
| [Region-partitioned topology](decisions/0001-region-partitioning.md) | A single global DB is sufficient on throughput but puts all patients' PII in one jurisdiction and makes any outage a global outage — fails residency and isolation, not scale. |
| [Slots computed on read, never materialized](decisions/0002-computed-on-read-slots.md) | A materialized slot table would need bulk regeneration (or reconciliation) on every availability edit, reintroducing the exact "edits shouldn't void bookings" problem §7 forbids. |
| [Global specialty registry, regional read replicas](decisions/0003-global-specialty-registry.md) | Fully synchronous global reads reintroduce a cross-region dependency this architecture is built to avoid; fully separate per-region registries lose the point of having a canonical, cross-country concept at all. |
| [Transactional outbox for notifications](decisions/0004-transactional-outbox.md) | A broker/CDC pipeline (Kafka+Debezium) is at-least-once by construction but is infrastructure sized for a scale and ops team this system doesn't have. |
| [Suggestion endpoint takes a doctor-supplied unavailability window](decisions/0005-suggestion-unavailability-window.md) | The literal requirement ("list free slots") would suggest slots inside the doctor's own absence — the brief's own motivating scenario. |
| [tzdata updated via `tzupdater`, tracked independently of JDK upgrades](decisions/0006-tzdb-versioning.md) | Zone rules can change with weeks' notice; waiting for the next scheduled JDK patch release is too slow. |
| REST/JSON over HTTPS as the inter-tier protocol ([01-architecture.md §5](docs/01-architecture.md#5-inter-tier-protocol-13--justified-against-the-js-only-frontend-constraint)) | gRPC needs codegen + a grpc-web proxy for browsers; GraphQL needs a resolver layer — both impose backend-shaped tooling on a JS-only frontend team for no offsetting benefit here. |
| Modular monolith per region, not microservices per region ([01-architecture.md §3](docs/01-architecture.md#3-layers-within-a-region)) | Each region is small; splitting into independently deployed services adds coordination overhead (discovery, tracing, network hops) with no scaling justification at this size. |

## Suggested reading order

1. [`docs/01-architecture.md`](docs/01-architecture.md) — start here for the overall shape.
2. [`docs/02-data-model.md`](docs/02-data-model.md) — schema, indexes, and the two sequence diagrams (concurrent booking, token cancellation is in doc 01).
3. [`docs/03-domain-model.md`](docs/03-domain-model.md) — class diagram and the appointment lifecycle state machine.
4. [`docs/05-time-and-dst.md`](docs/05-time-and-dst.md) — read this even if pressed for time; it's the part of the problem most often gotten wrong.
5. [`docs/04-suggestion-algorithm.md`](docs/04-suggestion-algorithm.md) — the ranking algorithm and the fix for the broken literal requirement.
6. [`docs/06-future-patient-mgmt.md`](docs/06-future-patient-mgmt.md) — the phase-2 migration reasoning.
7. [`decisions/`](decisions/) — as referenced above; this is also where the three uncomfortable-but-honest points live as deliberate records rather than buried asides (throughput doesn't justify partitioning; the specialty registry's curation-vs-immediacy tension; the literal suggestion requirement is broken).

## Deliberately out of scope

- **Free-text/fuzzy specialty search.** The slot-search API takes a specialty *id* from a constrained, per-country list; resolving free text to an id is a separate concern that would sit in front of this API (§4).
- **The identity provider itself.** This design names where OIDC attaches (`01-architecture.md` §1, §5) and does not build or select one.
- **A GitHub Pages site or any static-site build.** Mermaid renders natively on github.com; a build pipeline adds a dependency and a failure mode (Mermaid not rendering without plugin configuration) for a six-document submission that doesn't need one (§15).
- **A full phase-2 patient-management system.** §11 asks for migration reasoning, not a phase-2 design — `06-future-patient-mgmt.md` stops at the seam and the migration path, and explicitly flags (rather than solves) the access-control question a durable patient identity raises.
- **A standing "time off" / vacation-calendar feature for doctors.** The suggestion endpoint's unavailability window (`decisions/0005`) is request-scoped by design; a persisted absence calendar is a larger feature this submission doesn't attempt.
- **Retroactive historical tzdata corrections.** `05-time-and-dst.md` §4 handles forward-looking zone rule changes; a government rewriting a *past* rule is called out as an explicit assumption, not handled.
- **Load testing / benchmarking.** This is a design submission; the p99 target is addressed by index and cache design (`02-data-model.md` §3) and reasoned about, not measured against a running system.
- **Code.** Per the brief's own instruction — design documents only.

## Checklist mapping (§14)

| Checklist item | Addressed in |
|---|---|
| DST gap and overlap handling | [`05-time-and-dst.md` §3](docs/05-time-and-dst.md) |
| tzdb versioning and update strategy | [`05-time-and-dst.md` §4](docs/05-time-and-dst.md), [`decisions/0006`](decisions/0006-tzdb-versioning.md) |
| Computed-on-read meeting p99 < 300ms for 20 doctors | [`02-data-model.md` §3](docs/02-data-model.md), [`decisions/0002`](decisions/0002-computed-on-read-slots.md) |
| Concurrent booking of the same slot, and where enforced | [`02-data-model.md` §2 and §4](docs/02-data-model.md) (sequence diagram) |
| Onboarding a country with no canonical mapping yet | [`02-data-model.md` §2](docs/02-data-model.md), [`decisions/0003`](decisions/0003-global-specialty-registry.md) |
| What happens to booked appointments when availability changes | [`02-data-model.md` §2](docs/02-data-model.md) ("appointment" table notes) |
| Authorization of patient cancellation without accounts | [`01-architecture.md` §9](docs/01-architecture.md) (sequence diagram), [`03-domain-model.md` §1](docs/03-domain-model.md) (`CancellationToken`) |
| Why the topology is region-partitioned despite the load | [`01-architecture.md` §2](docs/01-architecture.md), [`decisions/0001`](decisions/0001-region-partitioning.md) |
| Why notification dispatch cannot be synchronous with booking | [`01-architecture.md` §7](docs/01-architecture.md), [`decisions/0004`](decisions/0004-transactional-outbox.md) |
| The appointment lifecycle state machine | [`03-domain-model.md` §2](docs/03-domain-model.md) |
| Migration path to patient management | [`06-future-patient-mgmt.md` §5](docs/06-future-patient-mgmt.md) |

## Repository layout

```
README.md              ← this file
requirements.md         ← the source design brief, unmodified
docs/
  01-architecture.md
  02-data-model.md
  03-domain-model.md
  04-suggestion-algorithm.md
  05-time-and-dst.md
  06-future-patient-mgmt.md
decisions/
  0001-region-partitioning.md
  0002-computed-on-read-slots.md
  0003-global-specialty-registry.md
  0004-transactional-outbox.md
  0005-suggestion-unavailability-window.md
  0006-tzdb-versioning.md
```
