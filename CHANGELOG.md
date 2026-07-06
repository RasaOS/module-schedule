# CHANGELOG — `rasa.module.schedule`

Reverse-chronological. Each entry is a version bump.

---

## 0.1.0 — 2026-07-05

Initial shell — spine + spec + adapter seam + ledger template; skills
deferred per extract-after-proof; account-handle spine threaded.

The **bookings-as-records** module (the calendar face), one of six
business-ops shells. Ships:

- **`content/schedule-rules.md`** — the portable spine: the **Booking**
  record (id, account?, title, type, start, end, all_day, location,
  attendees, status, reminder, notes), the `tentative → confirmed → done |
  cancelled` lifecycle (cancel-over-delete; the human confirms), the
  date-management discipline (absolute ISO-8601 timezone-qualified datetimes;
  relative-resolved-at-capture; duration = end − start), and the boundaries.
- **`content/README.md`** + **`content/BUILD_PLAN.md`** — the author-time
  spec: entity model, lifecycle, the seam, the deferred skills (M-1..M-3),
  the soft cross-references, and the honest-scope deferral.
- **`seed/schedule-gate.md.template`** — the adapter seam (working hours,
  timezone, booking types, reminder policy); placeholder-free, honest
  defaults + examples.
- **`seed/schedule/SCHEDULE.md.template`** — the live, project-owned booking
  ledger with example rows + a legend.

**Account spine.** Every Booking references its customer by the shared
`@acct-<slug>` handle (optional here — internal bookings omit it), the same
handle `leads` / `contracts` / `invoices` / `field-log` use. No accounts
registry seeded — the account hub is canon task SA-032.

**Boundaries.** Owns the records, NOT the generation engine — availability /
recurrence / conflict resolution is `rasa.module.scheduler` (soft ref).
External calendar sync (Google/Outlook/ICS) is a design-only Connections
(SA-031) reference, not implemented. Reminder delivery is out of scope
(records intent only).

**Deferred (extract-after-proof).** The `/book`, `/schedule`, `/remind`
skills are NOT built — gated on a real consumer declaring the module in its
`requires.elements[]` (the `module.tasks` precedent).

Forked from `rasa.module.core`; stripped the domain-core scaffold
(SHAPE.md / agents / rules / skills / output-style-enforcement, the
CLAUDE + output-style seed templates). `bin/check-manifest` GREEN.
