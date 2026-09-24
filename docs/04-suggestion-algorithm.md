# 04 — Suggested Alternative Slots (§9, endpoint 3)

## 1. The flaw in the literal requirement, and why it matters

Read literally, "a doctor requests 10 suggested alternative slots for one of her future appointments" says nothing about *excluding* the very reason she's asking. The motivating scenario in the brief is a doctor who is out sick on Tuesday and wants her Tuesday patients rebooked — but her own free-slot computation for Tuesday is exactly as free as any other day, because nothing in the domain model yet knows she's unavailable that day beyond her already-committed appointments. A naive implementation would happily suggest other Tuesday slots inside the same absence that triggered the request.

The fix fixed by this brief (§9, "Endpoint contracts") is to make the endpoint **accept a doctor-supplied unavailability window** that suggestions must avoid, in addition to her declared recurring availability and existing bookings. This is treated as a deliberate decision record rather than folded silently into the algorithm — see `decisions/0005-suggestion-unavailability-window.md`.

## 2. Inputs

- `originalAppointment` — the appointment being rescheduled: `doctorId`, `startInstantUtc`, `originTimezoneId`.
- `doctorUnavailability` — zero or more time windows the doctor supplies with the request (e.g. "all of Tuesday 2017-11-07"), expressed as local wall-clock windows in her timezone.
- `maxResults` — fixed at 10 per the brief.

## 3. Algorithm

```
function suggestAlternatives(originalAppointment, doctorUnavailability, maxResults=10):
    doctor = loadDoctor(originalAppointment.doctorId)
    today  = now() in doctor.timezoneId
    horizonEnd = today + doctor.bookingHorizonDays

    # Step 1 — reuse the same slot-derivation engine as search (02-data-model.md §3),
    # scoped to exactly one doctor, so cost is bounded the same way search's is.
    freeSlots = SlotCalculator.freeSlots(doctor, range(today, horizonEnd))
        # already excludes: slots outside declared availability,
        #                   slots already held (existing appointments),
        #                   slots beyond the booking horizon

    # Step 2 — exclude the doctor's declared unavailability window(s).
    # This is the fix for the flaw in §1: without it, freeSlots on the very day
    # she's out would still look "free" because nothing else marks that day unavailable.
    candidates = [s for s in freeSlots if not overlapsAny(s, doctorUnavailability)]

    # Step 3 — never suggest a slot that is already held is satisfied by construction
    # (freeSlots already subtracts existing appointments), but if the request also
    # supplies a "hold list" from a concurrently-open UI session, filter defensively:
    candidates = [s for s in candidates if s != originalAppointment.slot]

    # Step 4 — rank by the stated objective, lexicographically, in the order
    # the brief states its bullets (see §4 below for why lexicographic, not weighted).
    originalLocal = originalAppointment.startInstantUtc.atZone(doctor.timezoneId)
    ranked = sortBy(candidates, key = s -> (
        abs(daysBetween(s.localDate, originalLocal.date)),      # 1. minimize displacement
        s.localDate.dayOfWeek != originalLocal.dayOfWeek,        # 2. prefer same weekday (0 = match)
        abs(minutesBetween(s.localTime, originalLocal.time)),    # 2. prefer same hour of day
        s.startInstantUtc                                        # tiebreaker: chronological, deterministic
    ))

    return ranked.take(maxResults)
    # Fewer than maxResults is a valid response, not an error, if the doctor
    # genuinely has fewer than 10 qualifying free slots left in her horizon.
```

## 4. Why lexicographic ranking, not a weighted score

A weighted-sum score (`score = w1·dayDelta + w2·hourDelta - w3·weekdayMatch`) requires choosing weights that trade off, say, "3 days away at the same hour" against "1 day away at a very different hour" — and any such weights are an arbitrary design decision the brief doesn't hand us. Lexicographic ordering avoids inventing that tradeoff: it applies the brief's own stated priority order directly — displacement first (fewest calendar days away wins outright, regardless of hour), then weekday match as a tiebreaker among equally-displaced candidates, then hour-of-day closeness as a further tiebreaker, then a final chronological tiebreaker purely for deterministic, reproducible output (so the same inputs always produce the same ordering, which matters for testing and for a doctor re-running the same request).

## 5. Concurrency note

Suggestions are **proposals, not holds** — nothing in this endpoint reserves a slot. Between a suggestion being shown and the doctor or patient acting on it (via the rescheduling endpoint), another booking could legitimately claim that slot. That's handled exactly the same way as any other booking race: the rescheduling endpoint attempts the insert and relies on the `UNIQUE (doctor_id, start_instant_utc)` constraint (§6, `02-data-model.md` §4) to reject it if it's gone, rather than this endpoint trying to hold a lock on a slot that doesn't exist as a row to lock in the first place.
