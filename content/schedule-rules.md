# Schedule Rules

The portable **bookings-as-records** spine that `rasa.module.schedule`
installs at `.claude/schedule-rules.md`. It covers the one record this
module owns (the **Booking**), its canonical field set, its lifecycle, the
shared account-handle convention, the boundary against adjacent modules, and
the date-management conventions every booking obeys. **Read this file when
adding a booking, moving a date, setting a reminder, or asking "what's on the
calendar and what state is it in?"**

This file is **Element-owned** — it refreshes on upgrade. It deliberately
does **not** decide the one thing that varies per project:

- **Your working hours, timezone, booking types, and reminder policy.**

That lives in the project-owned **`.claude/schedule-gate.md`** (the schedule
analogue of the task module's done-gate and the notes module's notes-canon).
This file references the seam; it hardcodes none of it.

## What this module owns — and what it does not

This module owns the **calendar face**: a project-owned ledger of bookings as
plain, ordered, git-versioned markdown records. It is deliberately
**lightweight** — a booking is a record, not a scheduling computation.

It is **NOT** the generation engine. Anything that *computes* a schedule —
availability solving, recurrence expansion, conflict resolution across many
resources, optimal slot-finding — belongs to `rasa.module.scheduler` (a soft
sibling; see the boundary section). If you find yourself writing a solver
here, stop: the record layer stays dumb on purpose. Complexity is the
engine's job; this module just holds what was booked.

## The Booking — the one record

Every entry in `schedule/SCHEDULE.md` is a **Booking**. Its canonical field
set:

| Field | Required | Shape | Notes |
|---|---|---|---|
| `id` | yes | `BKG-NNN` monotonic | Never reused, never renumbered. |
| `account` | optional | `@acct-<slug>` | The customer handle (see the shared spine below). Omit for internal bookings with no customer. |
| `title` | yes | one line | What the booking is ("Kickoff call", "Site visit"). |
| `type` | yes | from the seam's list | e.g. `call \| meeting \| visit \| appointment`. Defaults to `general` if the seam is empty. |
| `start` | yes | ISO 8601 datetime | `2026-07-08T14:00:00-04:00`. Absolute, timezone-qualified. |
| `end` | optional | ISO 8601 datetime | Omit when `all_day` or when duration is implied by `type`. |
| `all_day` | optional | `true \| false` | When `true`, `start`/`end` carry a date only (`2026-07-08`) and time is ignored. |
| `location` | optional | one line | Physical or URL ("Zoom", "123 Main St"). |
| `attendees` | optional | list | Names/handles; freeform. |
| `status` | yes | lifecycle state | See below. |
| `reminder` | optional | ISO datetime or offset | Absolute (`2026-07-08T13:00:00-04:00`) or offset (`-1h`, `-1d`) resolved against `start`. Policy lives in the seam. |
| `notes` | optional | freeform | Anything else. |

A record with only the required fields is valid. The optional fields are
there when the project needs them, not a checklist to fill.

## The lifecycle — four states

A booking is always in exactly one state:

```
tentative  ──confirm──▶  confirmed  ──occur──▶  done
    │                        │
    └──────── cancel ────────┴──▶  cancelled
```

| State | Meaning |
|---|---|
| **tentative** | Proposed but not agreed. A hold, a "pencil it in." Nothing depends on it yet. |
| **confirmed** | Agreed and on the calendar. Reminders fire from this state. |
| **done** | The booking happened. Terminal. |
| **cancelled** | The booking will not happen. Terminal. Never delete a cancelled booking — the record that it *was* booked is the point. |

Rules for the lifecycle:

- **`tentative` and `confirmed` are the only live states.** `done` and
  `cancelled` are terminal — a booking never leaves them.
- **Cancel over delete.** A cancelled booking stays in the ledger with
  `status: cancelled`. Deleting loses the history that it existed.
- **The human confirms.** Moving `tentative → confirmed` is a ratification,
  not an inference — surface the candidate, let the person confirm. (Mirrors
  the substrate-wide "never auto-commit; the human commits" convention.)
- **`done` is set when the booking's `start` has passed and it occurred** —
  it is not automatic; a passed `confirmed` booking that didn't happen is
  `cancelled`, not `done`.

## Customer / Account reference (shared spine)

> **Customer / Account reference (shared spine).** Every record this module
> owns references its customer by a stable **account handle**:
> `account: @acct-<slug>` in the record's frontmatter (for example
> `@acct-acme`). The handle is an opaque, stable join key. This module **never
> invents or mutates the account's own record** — the account's name,
> contacts, address, and billing details live on the **account hub**, which
> RasaOS does not yet have as a first-class primitive (tracked as canon task
> **SA-032 — engagement-hub primitive**). Until the hub lands, treat the
> handle as the join key and, if a project-owned account ledger is mounted,
> resolve display names from it; **do not seed a competing accounts registry
> here.** Every sibling business-ops module (`leads`, `schedule`, `contracts`,
> `invoices`, `field-log`) uses this same `@acct-<slug>` handle, so records
> join across modules today by shared key and refactor to the canonical hub
> FK — with no change to the records themselves — when SA-032 resolves.

For this module specifically: `account` is **optional** on a Booking. An
internal booking (a team standup, a personal block) has no customer and omits
the handle. A customer-facing booking carries `@acct-<slug>` and joins to the
same customer that a `leads`, `contracts`, or `invoices` record references —
that shared handle is what lets "everything for @acct-acme" span the modules.

## Date-management conventions

The lightweight date discipline every booking obeys:

- **Absolute datetimes, always ISO 8601, always timezone-qualified.** Write
  `2026-07-08T14:00:00-04:00`, never `2pm Tuesday`. The project timezone is
  declared in the seam; a booking may override it with its own offset (a
  booking in another city keeps that city's offset).
- **Relative dates are resolved at capture, then stored absolute.** "Next
  Thursday" is a convenience for *entering* a booking, never a stored value —
  resolve it against today, write the absolute datetime, and the ledger
  stays unambiguous forever.
- **Duration is `end − start`.** There is no separate duration field. Omit
  `end` for `all_day` bookings or where `type` implies a standard length
  (the seam may declare default lengths per type).
- **All-day bookings carry a date, not a time.** `all_day: true` means
  `start`/`end` are `YYYY-MM-DD`; any time component is ignored.
- **Reminders are absolute or an offset.** `-1h` / `-1d` resolve against
  `start`; an absolute datetime is taken as-is. Whether a reminder actually
  *fires* — and how — is not this module's job; it records the intent. The
  project's reminder policy (lead times, channels) lives in the seam.

## The adapter seam — `.claude/schedule-gate.md`

The one per-project thing. It holds what genuinely varies per project:

1. **Working hours** — the hours bookings normally fall within (informational;
   the module does not enforce them, but skills and a human reviewer use them).
2. **Timezone** — the project's default timezone for bookings without an
   explicit offset.
3. **Booking types** — the project's `type` vocabulary (the `call | meeting |
   visit | ...` list), optionally with default durations.
4. **Reminder policy** — default lead times and the channel/mechanism (or a
   pointer to it). This module records reminder *intent*; it does not deliver.

The seam ships with honest defaults. This file references it; it hardcodes
none of it. A skill that needs "what's a valid type here?" or "what timezone
do bare datetimes mean?" reads the seam.

## Ledger conventions

- One `schedule/SCHEDULE.md`, git-versioned, project-owned (seeded
  skip-if-exists). It grows over the project's life.
- Markdown, human-first. The ledger must read cleanly top-to-bottom to a
  person — the retrievability is a person reading in date order, not a query.
- **`BKG-NNN` monotonic, never reused.** Order the ledger by `start` (or
  keep it append-and-sort); either way a human should be able to scan
  "what's coming up."
- **Dates absolute, timezone-qualified.** Never relative, never bare.
- **Cancel or mark done; never delete.** The point of the ledger is that the
  booking history survives.

## Soft cross-references (degrade gracefully)

This module stands alone, but plays well with siblings when mounted:

- **`rasa.module.scheduler`** — the **generation engine**. When a real
  scheduling computation is needed (availability, recurrence, conflict
  resolution), that is the scheduler's job; this module holds the resulting
  bookings as records. Soft sibling — present-if-mounted, never a dependency.
- **External calendar sync** (Google / Outlook / ICS) — a **design-only**
  soft reference to Connections (canon task **SA-031**). Two-way sync with an
  external calendar is NOT implemented here and is NOT this module's job; the
  ledger is the source of truth locally, and any sync is a future
  Connections-mediated concern.
- **The sibling business-ops modules** (`leads`, `contracts`, `invoices`,
  `field-log`) — join to the same customer via the shared `@acct-<slug>`
  handle (see the account spine). Present-if-mounted; the handle is the only
  coupling.

Do **not** harden any of these into a `requires.elements[]` dependency.
