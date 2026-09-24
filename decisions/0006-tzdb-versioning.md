# 0006 — tzdb version tracking and update mechanism

## Status
Accepted

## Context
§3 of the brief requires the submission to state which tzdb version is depended on and how it's updated, and explicitly warns against treating zone rules as static — political changes to zone rules can arrive with only weeks of notice.

## Options considered

**A — Whatever tzdata ships with the JVM at deployment, updated only on JDK upgrades.**
Rejected as the sole mechanism: JDK patch release cadence is not tied to IANA tzdata release cadence, so a zone rule change announced with a few weeks' notice could easily predate the next scheduled JDK upgrade, producing incorrectly computed slots for that zone in the gap.

**B — Vendor a separate timezone library (e.g. an external tzdata-only dependency) instead of relying on the JVM's bundled copy.**
Rejected: adds a second source of truth for timezone rules alongside `java.time`'s own, with its own update cadence to track and its own risk of drifting out of sync with the JDK's assumptions about `ZoneRules` semantics — more moving parts for the same underlying data.

**C — Use the JDK's own `tzupdater` tool to patch the running JVM's embedded tzdata independently of JDK version upgrades, on a tracked schedule (chosen).**
`tzupdater` is JDK-provided specifically for this problem: applying a newer IANA tzdata release to an already-installed JDK without waiting for that JDK's next patch release.

## Decision
Option C, operationalized as:
- Subscribe to IANA tz-announce (or an equivalent release-tracking feed) for new tzdata releases.
- Apply updates via `tzupdater` on a runbook cadence tied to actual announcements, not a fixed calendar schedule — because the whole point is that announcements arrive irregularly and sometimes urgently.
- Each region updates independently (its JVMs are its own operational concern, per `0001-region-partitioning.md`), so a region can be patched without needing to coordinate a global rollout.
- A regression test suite pins expected UTC offsets for known transition dates in every zone the system actually serves, so CI fails loudly if a deployed environment's tzdata is stale or a manual patch was missed, rather than the system silently mis-computing slots.

## Consequences
- Because availability is stored as wall-clock + zone id rather than a precomputed offset (§3, `05-time-and-dst.md` §1), applying a tzdata update requires **no data migration** — every future doctor's slots are correctly re-derived the next time they're computed, which is the whole reason that storage choice was made.
- Already-booked appointments (stored as fixed UTC instants) are unaffected by a tzdata update, which is correct for the common case (a future rule change shouldn't retroactively move a promise already made) but is explicitly noted as not covering the rare case of a government retroactively rewriting a *past* zone rule — out of scope, called out rather than silently assumed away.
