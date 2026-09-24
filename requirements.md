Usted
# Design Brief: Worldwide Doctor Appointment System

You are producing an **architecture and design submission** for the system described below. This brief refines the original requirements: every ambiguity in the source document has been resolved into an explicit decision, and the decisions are binding. Where a decision is deliberately left to you, this brief says so.

Produce design documents, not code.

---

## 1. Problem

A worldwide appointment system for doctors, operating 24/7. Two actors: **doctors** and **patients**.

- Doctors have one or more specialties and a city of operation.
- A doctor defines her working time by weekday and hour (e.g. Mon–Fri 08:00–14:00 and 16:00–18:00, Sat 10:00–14:00) and a slot unit in minutes (e.g. 30 or 60).
- An appointment commits one slot of one doctor to one patient, identified by email address.
- Patients search for free slots by date, city and specialty (e.g. a traumatologist on 2017-11-04 in Boston, MA, USA).

## 2. Fixed constraints

**Stack.** Java on the backend; JavaScript on the frontend. The backend team are Java experts; the frontend team writes JavaScript only. Treat this as a hard constraint — a design requiring frontend engineers to write Java is a failed design.

**Two frontend applications**, against one backend API:
1. A public, anonymous **patient booking app**.
2. An authenticated **doctor portal**.

They are separate because the authentication requirements differ (§5). Do not bundle them.

**Scale assumptions.** Size the design against these figures; do not invent different ones.
- ~50,000 doctors
- ~5,000,000 appointments per year
- Peak ~200 slot searches/second
- Strongly read-heavy: searches vastly outnumber bookings

**Topology: region-partitioned, multi-region.** Each region authoritatively owns the doctors and appointments for the geographies it serves.

The driver for partitioning is **data residency, regional failure isolation, and end-user latency — it is not throughput.** At the scale above, a single well-indexed relational database would handle the entire global load. State this honestly in your submission. The argument for partitioning is that an appointment record pairs an email address with a medical specialty (sensitive data under GDPR and equivalent regimes), that losing one region must degrade one geography rather than the system (the 24/7 requirement), and that users are globally distributed. Each individual region is small and runs on modest infrastructure.

Partitioning is cheap here because **appointments are inherently geographically local**: a patient in Boston books a doctor in Boston. There are no cross-region transactions in this system. Only the specialty registry (§4) is globally shared, and it is read-heavy and tolerates staleness.

## 3. Time semantics

Time is the most error-prone part of this system. The following rules are binding.

**Recurring availability** is stored as **wall-clock time plus an IANA timezone id** on the doctor. It is not stored as a UTC offset. A doctor who says "I start at 08:00" means 08:00 local, permanently — her schedule must survive daylight-saving transitions unchanged.

**Appointments** are stored as a **UTC instant plus the originating IANA zone id**. The instant makes "is this in the past?" unambiguous; the zone id preserves the local meaning for display and for month/date bucketing.

**Query interpretation.** A requested date (`2017-11-04`) and a requested month resolve in the **clinic's timezone** — never the caller's, never UTC.

**DST policy** must be implemented explicitly, not left to library defaults:
- Slots falling in a **nonexistent** local time (spring-forward gap) are **skipped**.
- Slots falling in a **repeated** local time (fall-back overlap) are offered **once**, at the **first** occurrence.

**Timezone database.** State which tzdb version you depend on and how it is updated. Zone rules change by political decree, sometimes with weeks of notice; a design that treats them as static is wrong.

## 4. Specialties

Specialties cannot be enumerated in advance — they differ by country.

**Model.** An internal **canonical specialty concept registry**, globally shared, with **per-country localized labels and aliases**. Seed it from an existing standard (SNOMED CT practitioner specialties or the HL7 PractitionerRole value set) but do not bind the system to one — external standards have gaps and release cycles you cannot onboard a country around.

**Curation and the "unknown in advance" requirement.** These two pull against each other, and your submission must address the tension rather than hide it. A canonical registry implies curation; curation implies latency; the requirement says a country's specialties are not known ahead of time. Resolution: a country may add a specialty **immediately**, as **locally valid but unmapped**. It is usable for doctor profiles and patient search within that country from the moment it exists. An administrative curation step later links it to a canonical concept. Onboarding a country must never block on a curator.

**A doctor may hold multiple specialties.**

**Search input.** The slot-search API accepts a specialty **id**, selected by the patient from a constrained list of that country's specialties in their language. Free-text and fuzzy specialty resolution is **out of scope** — note it as a separate concern that would sit in front of the same endpoint. Rationale: every use case scopes search to a city, a city sits in exactly one country, so search never crosses specialty vocabularies, and an id-based contract makes the lookup exact.

## 5. Identity and authorization

**Patients have no accounts.** A patient is an email address, exactly as the source requirements state.

**Cancellation is authorized by a signed, expiring token** delivered by email. Without this, anyone able to guess an appointment identifier could cancel a stranger's medical appointment. Treat the token flow as a designed feature, not a detail.

**Doctors authenticate.** Name the point where an identity provider attaches; do not design the identity provider.

## 6. Booking consistency

**At most one holder per `(doctor, slot start instant)`**, enforced by a **database uniqueness constraint** — not by application-level checking. Because the doctor and the appointment live in the same region, this is a single-shard write, so strict correctness is nearly free. Connect this to the partitioning decision in §2.

**No cross-doctor constraint on a patient.** One email may hold overlapping appointments with different doctors; enforcing otherwise would require a global index keyed on an email address, which is expensive and wrong for a system with no patient accounts. The same patient double-booking the *same* doctor slot is prevented for free by the uniqueness constraint above.

## 7. Slots

**Slots are computed on read** from the doctor's availability rules, minus her existing appointments. There is no materialized slot table. This is a fixed decision — see the performance requirement in §10.

**Consequence: appointments store their own absolute time and never reference a slot definition.** A slot is a derived view, not an entity.

**Booking horizon** is per-doctor and configurable, defaulting to **90 days**. No slots are offered beyond it. The horizon is a clinical-practice decision — a dermatologist booking six months ahead and a walk-in clinic booking two weeks ahead are both normal.

**Availability changes never void existing appointments.** If a doctor narrows her hours or changes her slot unit such that booked appointments now fall outside her declared availability or off the slot grid, those appointments **remain valid**. The system reports the conflicts to the doctor for manual resolution. An appointment is a commitment to a named person; the system must not silently void one because of an administrative edit.

## 8. Appointment lifecycle

Define an explicit **state machine**. Keep *cancelled*, *elapsed* and *no-show* distinct — collapsing them makes the lifecycle ambiguous and is a common failure in this problem.

**Cancellation** is permitted at any time **before** the appointment start and **rejected after**. There is no cutoff window.

## 9. Endpoints

### From the original requirements
1. A patient books an appointment with a given doctor.
2. A patient cancels an appointment.
3. A doctor requests **10 suggested alternative slots** for one of her future appointments.
4. A patient lists free doctor slots for a given date, city and specialty.
5. A doctor lists her appointments for a given month.

### Identified gaps — additions, not scope creep
State clearly in your submission that these were identified as gaps in the original use-case list.

6. **Doctor onboarding.** The requirements describe a doctor defining her working time and slot unit, but list no endpoint that writes it.
7. **Rescheduling.** Use case 3 produces suggested alternatives with no way to act on them, making the feature a dead end. Rescheduling is an **atomic** cancel-and-rebook, and is the one flow where the patient must be notified.
8. **Doctor-initiated cancellation.** Only patient cancellation is listed, yet the sick doctor is the document's own motivating example.

### Endpoint contracts fixed by this brief

**Free-slot search (4).** Results are **grouped by doctor** and **paginated over doctors**, with a hard cap on doctors per page. Do not paginate over individual slots: establishing a global ordering of slots requires evaluating every doctor in the city, which defeats pagination and contradicts §7.

**Monthly appointments (5).** Active appointments only. Month boundaries resolve in the clinic's timezone.

**Suggested alternatives (3).** Draw from the **same doctor's own free slots**, and accept a **doctor-supplied unavailability window that suggestions must avoid**. This matters: the motivating scenario is a doctor who is out on Tuesday, so her own Tuesday slots are useless — a naive reading of the requirement returns suggestions inside the very absence that triggered the request. Call out that flaw and why the unavailability window fixes it.

Objective for ranking, to be stated explicitly in the design rather than left implicit in code:
- Minimize the patient's displacement from the original appointment time.
- Prefer the same weekday and same hour of day.
- Never suggest a slot that is already held.
- Respect the doctor's booking horizon and the declared unavailability window.

## 10. Non-functional requirements

**Slot search performance: p99 under 300ms for a page of 20 doctors.** Since §7 fixes computed-on-read derivation, this is the load-bearing performance requirement of the design. Explain how you meet it — candidate mechanisms include per-doctor caching keyed on a rules version, an index on `(doctor, start instant)` over appointments, or a derived read model. Bounding work to page-size doctors per request is what makes derivation tractable: cost scales with the page, not with the size of the city.

**Notifications are in scope**, as an **asynchronous, event-driven** component. They are load-bearing: token-based patient cancellation (§5), doctor-initiated cancellation and rescheduling (§9) all require outbound email to function.

Notification dispatch must **never** participate in the booking transaction — an SMTP timeout must not roll back a confirmed appointment. Address delivery guarantees and at-least-once semantics (e.g. the transactional outbox pattern).

**Retention and erasure.** Retention periods for medical records are legally mandated and vary by jurisdiction, so the retention period is a **per-region property**, not a global constant — a direct consequence of the partitioning in §2. Automated purge on expiry. Additionally, support **erasure on request, authorized by verified email address**; note that email-as-identity makes this awkward and that the phase-2 patient entity (§11) makes it clean.

## 11. Forward compatibility: future patient management

Include a section reasoning about a future patient-management system layered onto email-as-identity. The substance is in the migration, not the intention. Address at minimum:

- Where a `Patient` aggregate attaches, and why email-as-identity does not block its introduction.
- **Claiming history**: how an account claims its historical appointments without letting someone acquire another person's medical history by typing their email address. You already have verified-ownership machinery from the §5 signed-token flow — use it.
- **Identity stability**: what happens when a patient changes email address, given that past appointments were keyed on the old one.
- **PII consequences**: the privacy profile changes once appointment history is durably associated with a person rather than scattered across an address, and how that interacts with §10 retention and erasure.

## 12. Deliverables

1. **Proposed architecture** for the whole system: modules and layers, backend technologies, frontend technologies, and the protocol between tiers.
2. **Data model** for the persisted schema: tables, relationships, keys, and the indexes that §10 depends on.
3. **Class diagram** for the backend domain model.
4. **High-level algorithm** for generating the list of suggested alternatives (§9).

## 13. Decisions left to you

These are deliberately open. Justify each.

- Java framework and runtime choices.
- **Inter-tier protocol.** Justify it against the JavaScript-only frontend constraint — that constraint is the source document's hint that the answer must not require frontend engineers to write Java.
- Persisted schema details and indexing strategy.
- Domain class structure and aggregate boundaries.
- The concrete suggestion-ranking implementation, against the objective fixed in §9.
- The mechanism by which computed-on-read slot derivation meets the p99 target in §10.

## 14. Checklist — your submission must explicitly address these

Each item below is a place where this problem is commonly under-designed. Do not leave any of them implicit.

- [ ] DST gap and overlap handling, per the policy in §3.
- [ ] tzdb versioning and update strategy.
- [ ] How computed-on-read derivation meets p99 < 300ms for 20 doctors.
- [ ] Concurrent booking of the same slot, and where the constraint is enforced.
- [ ] Onboarding a country whose specialties have no canonical mapping yet.
- [ ] What happens to booked appointments when a doctor's availability changes.
- [ ] Authorization of patient cancellation without patient accounts.
- [ ] Why the topology is region-partitioned when the stated load does not require it.
- [ ] Why notification dispatch cannot be synchronous with booking.
- [ ] The appointment lifecycle state machine.
- [ ] The migration path to patient management (§11).

## 15. Presentation

Deliver as a **GitHub repository**. Do **not** publish a GitHub Pages site: this submission is roughly six documents, which GitHub's own file browser plus a README index navigates adequately, and a static-site build adds two costs. A build pipeline for a six-document design submission reads as the same over-engineering instinct the submission is being judged against; and **Mermaid renders natively on github.com but not on GitHub Pages without explicit plugin configuration**, so the build step's default failure mode is that the class diagram and ERD — two of the four required deliverables — do not render.

### Layout

```
README.md ← reviewer's guide (see below)
requirements.md ← the original source requirements, unmodified
docs/
01-architecture.md ← modules/layers, technologies, inter-tier protocol
02-data-model.md ← ERD, schema, indexes with justification
03-domain-model.md ← class diagram
04-suggestion-algorithm.md ← ranking algorithm (§9)
05-time-and-dst.md ← time semantics, pulled out deliberately (§3)
06-future-patient-mgmt.md ← forward compatibility (§11)
decisions/
0001-region-partitioning.md
0002-computed-on-read-slots.md
... ← one ADR per significant decision
```

### README is a reviewer's guide, not an introduction

Reviewers are time-boxed and will read the front page plus two or three links. The README must contain:

- One paragraph of framing.
- A table of significant decisions, each with a one-line "why not the alternative".
- A suggested reading order.
- An explicit statement of what was deliberately placed out of scope — otherwise a deliberate omission is read as an oversight.
- A table mapping each checklist item in §14 to where it is addressed.

### Decision records

Maintain `decisions/` as short ADRs: context, options considered, decision, consequences. This is where rejected alternatives become visible without cluttering the main documents, and it is the right place for the uncomfortable-but-honest points this brief requires — that partitioning is not justified by the stated throughput (§2), that the canonical specialty registry conflicts with "specialties cannot be known in advance" (§4), and that the literal reading of the suggestion requirement is broken (§9). Those land as deliberate decision records; they read as hedging if buried as asides.

**Pull time and DST into its own document.** It cuts across every other document, and a dedicated file signals awareness of where this problem is actually hard.

### Diagrams

Use **Mermaid**, committed as text — diffable, no asset pipeline, renders on GitHub:

- `classDiagram` for the domain model
- `erDiagram` for the persisted schema
- `flowchart` for the architecture

Additionally provide **sequence diagrams** for two flows where the interesting behaviour is temporal and prose handles it poorly: **concurrent booking of the same slot** (§6), and **token-authorized patient cancellation** (§5).

If auto-layout makes the architecture diagram unreadable, hand-lay it and commit **both** the exported image and its editable source. Never commit an image without its source.

### Format fallback

Confirm whether the submission instructions mandate a specific format. A repository link can be unreachable for a reviewer behind a corporate firewall or without a GitHub account. If that is a risk, keep markdown as the source of truth and generate a **single PDF as a secondary artifact** — never the reverse.
