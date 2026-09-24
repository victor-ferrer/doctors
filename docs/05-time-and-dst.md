# 05 — Time and DST

Pulled into its own document deliberately (§15): time semantics cut across availability, search, booking, and the suggestion algorithm, and this is the part of the problem where an implicit or "obvious" choice is usually the wrong one.

## 1. The two storage rules (§3), and why they're different from each other

| | Stored as | Why |
|---|---|---|
| **Availability rule** | wall-clock local time + IANA zone id (e.g. `08:00`–`14:00`, `Europe/Madrid`) | A doctor's stated hours are a standing local-time commitment. "I start at 08:00" must still mean 08:00 local after a DST transition — storing a UTC offset would silently shift her opening time twice a year. |
| **Appointment** | UTC instant + originating IANA zone id | A booked appointment is a specific, fixed point in physical time — "is this in the past?" must be answerable by comparing instants, with no timezone arithmetic. The zone id is retained *alongside* the instant purely so the appointment can still be displayed and bucketed in the patient's/doctor's local terms later. |

These are not the same rule applied twice — one is intentionally relative-to-a-clock, the other is intentionally fixed-to-an-instant, because the two things they represent (a recurring policy vs. a one-time commitment) have different correctness requirements.

## 2. Query interpretation

A requested date (`2017-11-04`) or month always resolves in the **clinic's** timezone (the doctor's `timezoneId`), never the caller's browser timezone and never UTC. Concretely: "doctor's appointments for November 2017" means `[2017-11-01T00:00, 2017-12-01T00:00)` evaluated in `doctor.timezoneId`, converted to a UTC instant range only to run the indexed query — the `appointment.local_date` generated column (`02-data-model.md` §2) stores exactly this pre-resolved local date so the query never has to redo the conversion per row.

## 3. DST policy — explicit, not library-default

Two cases, both **required by §3 to be handled explicitly** rather than trusted to whatever `java.time` does by default:

- **Gap (spring-forward).** A local time like `02:30` on the day clocks jump from `02:00` to `03:00` never occurs. **Policy: skip that candidate slot entirely** — it is not offered, not shifted.
- **Overlap (fall-back).** A local time like `01:30` occurs twice (once before, once after clocks are set back). **Policy: offer it once, at its first (earlier) occurrence.**

### Why "library default" is explicitly called out as insufficient
`java.time`'s `LocalDateTime.atZone(ZoneId)` has its own built-in resolution behavior for both cases (roughly: shift a gap forward by the gap's length rather than reject it; pick a specific offset for an overlap). Relying on that default silently is exactly what §3 forbids, for two reasons: first, the gap default (shifting forward) produces a slot at a time nobody asked for instead of skipping it, which is the opposite of this system's policy; second, even where a default happens to agree with policy, depending on undocumented default behavior means a future JDK change or a library swap could silently change booking behavior with no test ever having pinned down *why* it was correct. This design resolves both cases explicitly, one time zone rule lookup at a time:

```
function resolveCandidateSlot(localStart: LocalDateTime, zone: ZoneId) -> Instant | SKIP:
    rules = zone.getRules()
    transition = rules.getTransition(localStart)

    if transition != null and transition.isGap():
        return SKIP                                   # spring-forward: this local time doesn't exist

    if transition != null and transition.isOverlap():
        offset = transition.getOffsetBefore()          # deterministic: the earlier of the two valid
                                                         # offsets == the first chronological occurrence
    else:
        offset = rules.getOffset(localStart)            # ordinary, unambiguous local time

    return localStart.atOffset(offset).toInstant()
```

This is run once per candidate slot when generating a doctor's availability into concrete instants (inside `SlotCalculator`, `03-domain-model.md`), so the gap/overlap decision is made in exactly one place in the codebase, not re-derived ad hoc anywhere a slot is displayed or booked.

### Worked examples
- **Gap:** `America/New_York` on 2018-03-11, clocks jump `02:00 → 03:00`. A doctor with a 30-minute slot unit and a `02:00–04:00` availability window would nominally generate candidates at `02:00, 02:30, 03:00, 03:30`. `02:00` and `02:30` fall inside the gap and are **skipped**; the doctor's actual offered slots that day are `03:00, 03:30`.
- **Overlap:** `America/New_York` on 2018-11-04, clocks fall back `02:00 → 01:00`. A `01:00–02:00` window's candidate `01:30` is ambiguous (occurs at UTC-04:00 first, then again at UTC-05:00). Policy offers it **once**, resolved to the UTC-04:00 (earlier) instant — the doctor and patient both see one `01:30` slot, not two, and not zero.

## 4. tzdb version and update strategy

The JVM ships a bundled copy of the **IANA Time Zone Database (tzdata)**. Two facts make this a live operational concern rather than a one-time setup detail:

1. **Zone rules change by political decree**, sometimes with only weeks of notice (a country abolishing DST, moving a boundary, changing a transition date) — the brief is explicit that a design treating tzdb as static is wrong.
2. Because availability is stored as **wall-clock + zone id, not a precomputed offset** (§1 above), a tzdata update automatically and correctly re-derives every future doctor's slots the moment it's applied — no data migration is needed, which is one of the concrete reasons that storage choice was made rather than pre-storing offsets.

**Operational policy:**
- Track the **IANA tz-announce** mailing list / release feed for new tzdata releases.
- Apply updates using the JDK's own timezone-data updater (`tzupdater`), which patches the running JVM's embedded tzdata **without waiting for a full JDK patch release** — this is the mechanism that closes the gap between "a government announces a change with three weeks' notice" and "the next scheduled JDK upgrade."
- Treat this as a scheduled operational runbook item per region (each region's JVMs need the update independently — another small benefit of regions being operationally decoupled, per `decisions/0001-region-partitioning.md`), verified by an automated regression test suite that pins expected UTC offsets for a fixed set of known transition dates across the zones the system actually serves, so a missed or malformed tzdata update fails CI rather than silently mis-booking appointments.
- **Explicit scope limit:** this design does not attempt to handle a government retroactively changing a *past* zone rule (rare, but it has happened historically). Appointments already stored as fixed UTC instants are, by construction, unaffected by any tzdata change going forward — which is correct for future rule changes but is called out here as an assumption, not silently glossed over, for the retroactive case.
