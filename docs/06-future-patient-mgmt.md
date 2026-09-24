# 06 — Forward Compatibility: Future Patient Management

The brief asks for reasoning about the migration, not a speculative phase-2 design. This document stays at that level: where the seam is, and why crossing it later doesn't require breaking anything that exists today.

## 1. Where a `Patient` aggregate attaches

Two additive tables, no change to any existing one:

```sql
CREATE TABLE patient (
  id uuid PRIMARY KEY,
  primary_email citext NOT NULL,
  created_at timestamptz NOT NULL
);

CREATE TABLE patient_email (
  patient_id uuid REFERENCES patient(id),
  email citext NOT NULL,
  verified_at timestamptz NOT NULL,
  PRIMARY KEY (patient_id, email)
);

ALTER TABLE appointment ADD COLUMN patient_id uuid NULL REFERENCES patient(id);
```

`appointment.patient_email` (the existing column) is untouched — it remains the historical record of *what email was used to book*, which stays true and meaningful forever, regardless of whether it's later linked to an account. `patient_id` is nullable and simply absent on every appointment until claimed. Nothing about booking, search, or cancellation today reads or writes `patient_id`, so this is a zero-downtime, zero-behavior-change schema addition — email-as-identity doesn't block the introduction of a `Patient` aggregate because the aggregate was never required to replace the email column, only to optionally sit alongside it.

## 2. Claiming history without letting anyone type their way into someone else's records

This reuses the signed-token machinery from §5 (`01-architecture.md` §9, `03-domain-model.md` §1's `CancellationToken`/`TokenPurpose`) rather than inventing a second mechanism:

1. A person creates an account and asserts an email address they want to claim appointments for.
2. The system sends a signed, expiring token to that email, with `purpose = CLAIM` (distinct from `purpose = CANCEL` so a leaked or reused cancellation link from an old email can never be replayed as a claim, and vice versa — purpose is part of what's signed, not an incidental field).
3. Only on clicking that link — proving control of the mailbox, not merely knowledge of the address — does the system set `patient_id` on every `appointment` row where `patient_email` matches (case-insensitively, via the existing `citext` column).

This is the same trust argument as patient cancellation: **possession of the mailbox, not knowledge of the string, is what authorizes the action.** Typing a stranger's email into a "claim my history" form sends the proof-of-ownership challenge to *their* inbox, not to the requester — the claim simply never completes.

## 3. Identity stability across an email change

Two distinct sub-problems, both already solved by the above:

- **Going forward:** once a patient is authenticated (logged into a phase-2 account), new bookings are made with `patient_id` set directly at insert time — they no longer depend on which email string was typed, so a later email change doesn't affect them retroactively at all.
- **Historical appointments under the old email:** these were already linked via `patient_id` at claim time (§2), and that FK doesn't reference the email string — it's a durable identifier set once. Changing the account's primary email later doesn't require "re-finding" old appointments, because they were never going to be looked up by email again after being claimed.
- The `patient_email` table lets the account hold multiple **verified** aliases over time (the old address stays as a verified alias unless removed; a new address is added via the same claim-token flow). This means a stray future appointment booked anonymously under the *new* address before the patient logs in again can still be claimed and correctly attributed to the same `patient_id`, rather than starting a second, orphaned identity.

## 4. PII consequences for retention and erasure (§10)

Today, erasure-on-request is inherently awkward, and the brief says so explicitly: it means "verify control of an email address, then find and purge every row bearing that exact string" — which is precise but brittle (a patient who used two different email addresses over the years has two separate, disconnected erasure requests to make, and doesn't necessarily know that).

Once appointment history is durably associated with a `patient_id`:

- **Erasure and export become cleaner, not just different.** A request targets a `patient_id` and correctly reaches every historical appointment regardless of which email address was used to book each one at the time — something string-matching on a single email literally cannot do for a patient who changed addresses.
- **But a durable cross-appointment identity is a larger privacy surface than scattered rows.** A single key now aggregates a longitudinal medical history (which specialties, which dates, across possibly years) rather than leaving that history implicitly fragmented across whatever emails were used. This raises the stakes on *who can read* a `patient_id`'s full history and *why* — an access-control and audit-logging question that per-row, per-email data didn't pose as sharply. This design does not solve that access-control problem (it's phase-2 scope), but flags it here deliberately rather than letting the migration read as a pure improvement with no new considerations.
- **Retention purge and erasure logic stay per-region/per-jurisdiction** (`retention_policy` keyed by `country_code`, `02-data-model.md` §2) either way — the unit being purged shifts from "rows matching an email string" to "rows matching a `patient_id`, scoped to the appointments in this region," but the jurisdictional boundary the purge respects doesn't change.

## 5. Migration path, in order

1. **Additive schema** — `patient`, `patient_email`, nullable `appointment.patient_id`. No existing endpoint changes behavior.
2. **Claim-flow endpoints** — request a claim link, verify a claim token — built on the existing token infrastructure, not a new one.
3. **Optional authenticated booking** — once logged in, `patient_id` is set directly on new bookings. The anonymous, email-only flow is **not deprecated**; not every patient will want an account, and §5 of the brief's "patients have no accounts" baseline keeps working indefinitely alongside it.
4. **No silent backfill.** At no point does a batch job link `patient_id` to appointments by matching email strings without a claim token. That would quietly reintroduce exactly the "acquire someone else's history by typing their email" risk that the claim-token flow exists to prevent — the whole point of §2 above is that linkage only ever happens after proof of mailbox control, and a backfill job proves nothing.
