# Rasa · Module · Schedule

**Canonical name:** `rasa.module.schedule`
**Repo / folder:** `module-schedule`
**Kind:** `module` (canon Spec §6)
**Contract:** Element Contract v1.3.0
**Version:** 0.1.0 (spine + spec + seam + ledger; skills are the build phase)
**Status:** Local shell. Skill build gated on a real consumer declaring the module.

## What this is

The **bookings-as-records** module — a project-owned ledger of bookings
(calls, meetings, visits, appointments) with reminders and dates, as portable
git-versioned markdown. The **calendar face**, deliberately lightweight.

One of six business-ops module shells (`leads`, `schedule`, `scheduler`,
`contracts`, `invoices`, `field-log`) that give a domain or orchestrator a
coherent, joined set of operational records. They join across modules by a
shared customer handle (see the account spine).

## The record it owns — the Booking

One record: the **Booking**, in `schedule/SCHEDULE.md`.

| id | account? | title | type | start | end | all_day | location | attendees | status | reminder | notes |
|---|---|---|---|---|---|---|---|---|---|---|---|

Lifecycle: `tentative → confirmed → done | cancelled`. Cancel over delete;
the human confirms. Dates are absolute, ISO 8601, timezone-qualified. Full
field semantics in `content/schedule-rules.md`.

## Element- vs project-owned files

| File | Owner | Installs to | Policy |
|---|---|---|---|
| `content/schedule-rules.md` | Element | `.claude/schedule-rules.md` | file-replace (refreshes on upgrade) |
| `seed/schedule-gate.md.template` | **Project** | `.claude/schedule-gate.md` | skip-if-exists (the seam) |
| `seed/schedule/SCHEDULE.md.template` | **Project** | `schedule/SCHEDULE.md` | skip-if-exists (the live ledger) |
| `seed/rasa.lock.json.template` | Project | `.claude/rasa.lock.json` | init-only-with-sha |

`content/README.md` + `content/BUILD_PLAN.md` are author-time docs (opt-in;
not installed into consumers).

## The account spine

Every Booking references its customer by the shared **`@acct-<slug>`** handle
— the same handle `leads`, `contracts`, `invoices`, and `field-log` use, so
records join across the modules by shared key. The handle is optional here:
internal bookings (standups, personal blocks) omit it. This module never owns
the account's own record; the account hub is the missing primitive tracked as
canon task **SA-032**.

## Skills deferred to the build phase

The `/book`, `/schedule`, `/remind` skills are **not built** in v0.1.0. They
are gated on a real consumer declaring `rasa.module.schedule` in its
`requires.elements[]` (the `module.tasks` **extract-after-proof** precedent).
Their contracts + the gate are in `content/BUILD_PLAN.md` (M-1..M-3).

## Boundary (read this)

- **Not the generation engine.** Availability solving, recurrence, conflict
  resolution → `rasa.module.scheduler` (soft sibling). This module holds the
  resulting bookings as records; it computes nothing.
- **Not external calendar sync.** Google / Outlook / ICS is a **design-only**
  Connections (SA-031) reference, not implemented here.

## See also

- `content/BUILD_PLAN.md` — full spec, skill contracts, the deferral gate.
- `content/schedule-rules.md` — the installed spine.
- `~/rAI/rasa-os/elements/module-notes/` — the shell precedent this mirrors.
- `~/rAI/rasa-os/elements/module-scheduler/` — the generation-engine sibling.
- Canon Spec §6 — the `module` kind. `ELEMENT_CONTRACT.md` §7 — install policies.
- `~/rAI/rasa-os/elements/REGISTRY.md` — register this Element here on first ship.
