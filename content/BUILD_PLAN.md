# `rasa.module.schedule` — Specification & Build Plan

**Status:** v0.1.0 = spine + spec + seam + ledger template. The skills are
the build phase (M-1..M-3 below), deliberately gated.

This is the author-time specification. The installed spine is
`content/schedule-rules.md`; this file is the design record behind it.

---

## Why this module exists

One of six business-ops module shells (`leads`, `schedule`, `scheduler`,
`contracts`, `invoices`, `field-log`) that give a domain or orchestrator a
coherent, joined set of operational records. `schedule` owns the **calendar
face**: bookings as project-owned records.

The deliberate framing is **lightweight — bookings-as-records**. A booking is
a row in a git-versioned markdown ledger, not the output of a scheduling
computation. That split is the whole point: the *record* layer (this module)
stays dumb and durable; the *computation* layer (`rasa.module.scheduler`) is
where availability solving, recurrence, and conflict resolution live. Keeping
them separate is what lets `schedule` be a thin, honest shell that a project
can adopt without pulling in a solver it may not need.

## Non-goals (hard boundaries)

- **Not the generation engine.** No availability solving, recurrence
  expansion, conflict resolution, or slot-finding. That is
  `rasa.module.scheduler` (soft sibling). If a feature *computes* a schedule,
  it belongs there, not here.
- **Not external calendar sync.** Two-way sync with Google / Outlook / ICS is
  a **design-only** reference to Connections (canon task **SA-031**) and is
  **not implemented here**. The local ledger is the source of truth; any sync
  is a future Connections-mediated concern.
- **Not the account hub.** This module references customers by the shared
  `@acct-<slug>` handle and never owns the account's own record (name,
  contacts, billing). That is the engagement-hub primitive RasaOS doesn't yet
  have (canon task **SA-032**). See the account spine below.
- **Not a reminder delivery system.** A booking records reminder *intent*
  (when, and per the seam's policy). Actually *firing* a reminder — the
  channel, the daemon — is out of scope; the module holds the intent.

---

## Entity model

One project-owned record + one project-owned seam.

### `schedule/SCHEDULE.md` — the Booking ledger (the load-bearing file)

Append-and-order, one entry per booking. The canonical field set:

| Field | Required | Shape | Notes |
|---|---|---|---|
| `id` | yes | `BKG-NNN` monotonic | Never reused. |
| `account` | optional | `@acct-<slug>` | Customer handle; omit for internal bookings. |
| `title` | yes | one line | What the booking is. |
| `type` | yes | from the seam's list | `call \| meeting \| visit \| appointment \| …`; default `general`. |
| `start` | yes | ISO 8601 datetime | Absolute, timezone-qualified. |
| `end` | optional | ISO 8601 datetime | Omit for `all_day` / type-implied duration. |
| `all_day` | optional | bool | Date-only `start`/`end` when `true`. |
| `location` | optional | one line | Physical or URL. |
| `attendees` | optional | list | Freeform. |
| `status` | yes | lifecycle state | `tentative \| confirmed \| done \| cancelled`. |
| `reminder` | optional | ISO datetime or offset | `-1h` / `-1d` resolve against `start`. |
| `notes` | optional | freeform | Anything else. |

Example entry shape:

```
## BKG-007 — Kickoff call
- **Account:** @acct-acme
- **Type:** call
- **Start:** 2026-07-08T14:00:00-04:00
- **End:** 2026-07-08T14:30:00-04:00
- **Location:** Zoom
- **Attendees:** Chazz, Acme (2)
- **Status:** confirmed
- **Reminder:** -1h
- **Notes:** first call of the engagement.
```

Rules: `BKG-NNN` monotonic, never reused; dates absolute + timezone-qualified;
cancel or mark `done` — never delete.

### The lifecycle

```
tentative  ──confirm──▶  confirmed  ──occur──▶  done
    │                        │
    └──────── cancel ────────┴──▶  cancelled
```

`tentative` and `confirmed` are the live states; `done` and `cancelled` are
terminal. The human confirms (`tentative → confirmed` is ratification, not
inference). A passed `confirmed` booking that occurred is `done`; one that
didn't is `cancelled`, never silently `done`.

### `.claude/schedule-gate.md` (the seam) — the one per-project thing

See "The adapter seam" below.

---

## The account spine (shared across the six modules)

Every Booking references its customer by the stable `@acct-<slug>` handle —
the same handle `leads`, `contracts`, `invoices`, and `field-log` use. It is
an opaque join key; this module never owns or mutates the account's own
record. The account hub is the missing primitive tracked as canon task
**SA-032 — engagement-hub primitive**; until it lands, the handle *is* the
join, and records refactor to the canonical FK — with no change to the
records — when SA-032 resolves. The verbatim convention is in
`content/schedule-rules.md` → "Customer / Account reference (shared spine)".

For `schedule` specifically, `account` is **optional**: internal bookings
(standups, personal blocks) carry no handle; customer-facing ones do.

---

## The adapter seam — `.claude/schedule-gate.md`

The crux of the per-project variation, mirroring `module.tasks`' done-gate
and `module.notes`' notes-canon. It holds the four things that genuinely vary
per project:

1. **Working hours** — the hours bookings normally fall within (informational;
   used by skills + a human reviewer, not enforced).
2. **Timezone** — the default timezone for bookings without an explicit offset.
3. **Booking types** — the project's `type` vocabulary, optionally with
   default durations per type.
4. **Reminder policy** — default lead times + the channel/mechanism (or a
   pointer to it). The module records reminder *intent*; it does not deliver.

Why a seam and not fixed logic: working hours, timezone, the type vocabulary,
and reminder lead-times are exactly what differs between a law firm, a field
crew, and a sales team. The horizontal core is *the Booking record + its
lifecycle + the date discipline*; the vertical part is *the project's calendar
conventions*.

---

## Skills (the build phase — M-1..M-3)

Three skills, MVP-scoped. Each is a thin driver over the ledger + seam; the
discipline lives in `schedule-rules.md`, not duplicated per skill. **They are
deferred** — see the gate below.

### M-1 — `/book`
Add or confirm a booking. `/book <title> …` collects the required fields
(prompting from the seam's type list + timezone), assigns the next `BKG-NNN`,
and files a `tentative` (or, on explicit confirm, `confirmed`) entry in
`schedule/SCHEDULE.md`. Resolves any relative date to an absolute,
timezone-qualified datetime at capture. Confirming a `tentative` booking is a
ratification (surface, then the human confirms).

### M-2 — `/schedule`
The agenda view. `/schedule [today | week | upcoming | @acct-<slug>]` reads
the ledger and prints the relevant bookings in date order. Read-only. Default:
upcoming `confirmed` + `tentative` bookings.

### M-3 — `/remind`
Due reminders. `/remind` scans the ledger for bookings whose `reminder`
resolves to now-or-past and surfaces them, per the seam's reminder policy.
Read-only surfacing — it reports what's due; it does not deliver notifications
(delivery is out of scope, see Non-goals).

**Style/quality bar:** match the sibling modules' SKILL.md files — a crisp
operation list, honest error/hard-stop behavior, no invented plumbing. Author
`/book` as the reference skill, then `/schedule` + `/remind` in parallel
against it, gating each independently.

---

## The gate — when to build the skills

Per the RasaOS **extract-after-proof** precedent (the `module.tasks` rule),
hold the skill build until **one real consumer** wants it — a domain or
orchestrator that adds `rasa.module.schedule` to its `requires.elements[]`.
Building the skills before a declared consumer would repeat the
premature-abstraction trap the shell approach exists to avoid.

The natural first consumers are the same orchestrators that would pull the
other business-ops modules (an operations-style orchestrator managing customer
engagements), where `schedule` joins `leads` / `contracts` / `invoices` /
`field-log` on the shared `@acct-<slug>` handle.

---

## HONEST SCOPE / deferred

**What v0.1.0 actually is:** a shell. It ships the spine
(`content/schedule-rules.md`), this spec, the seam template
(`seed/schedule-gate.md.template`), and the ledger template
(`seed/schedule/SCHEDULE.md.template`). Nothing computes anything.

**Deliberately deferred:**

- **The `/book`, `/schedule`, `/remind` skills** — gated on a declared
  consumer (above). No `content/skills/` in v0.1.0.
- **Any scheduling computation** — availability, recurrence, conflict
  resolution. That is `module.scheduler`'s job, forever, not a later version
  of this module.
- **External calendar sync** — Google / Outlook / ICS, a design-only
  Connections (SA-031) concern; not built here.
- **Reminder delivery** — the module records reminder intent; firing is out
  of scope.
- **The account hub** — resolved when canon SA-032 lands; until then the
  `@acct-<slug>` handle is the join and this module seeds no accounts
  registry.

Overbuilding any of the above into v0.1.0 would be a failure. The shell is
the deliverable.

---

## Version plan

- **v0.1.0 (this)** — spine (`schedule-rules.md`) + spec (this file) + seam
  template + ledger template. No skills.
- **v0.2.0** — M-1..M-3 skills authored, once a real consumer declares the
  module. `bin/init` smoke-tested into that consumer.
- **v1.0.0** — the seam format + install shape locked after the record layer
  has run through at least one real vertical unchanged.
