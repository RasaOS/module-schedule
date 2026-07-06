# `rasa.module.schedule` — content

What this module ships and where it installs. This file is author-time
documentation (not installed into consumer projects).

## The one-liner

The **bookings-as-records** module: a project-owned ledger of bookings
(calls, meetings, visits, appointments) with reminders and dates, as portable
git-versioned markdown. The **calendar face** — deliberately lightweight. It
is NOT the generation engine; that is `rasa.module.scheduler`.

## What installs where

| Source | Installs to | Policy | What it is |
|---|---|---|---|
| `content/schedule-rules.md` | `.claude/schedule-rules.md` | file-replace | The spine — the Booking record, its lifecycle, the account spine, the date conventions, the scheduler/Connections boundary. Element-owned; refreshed on upgrade. |
| `seed/schedule-gate.md.template` | `.claude/schedule-gate.md` | skip-if-exists | **The seam** — working hours, timezone, booking types, reminder policy. Project-owned; the project fills it. |
| `seed/schedule/SCHEDULE.md.template` | `schedule/SCHEDULE.md` | skip-if-exists | The live booking ledger. Project-owned; grows over the project's life. |
| `seed/rasa.lock.json.template` | `.claude/rasa.lock.json` | init-only-with-sha | Connection-Contract lockfile, SHA-stamped at init. |

## What is NOT here yet

The `/book`, `/schedule`, `/remind` **skills** — they are the build phase.
See [`BUILD_PLAN.md`](BUILD_PLAN.md) for their contracts and the
extract-after-proof gate that holds them until a real consumer declares the
module in its `requires.elements[]`.

## The shape

Toolkit module, `requires.parent_kind: [domain, orchestrator]`. Pure
Element-layer convention — no kernel engine. The scheduling *computation*
lives in `module.scheduler`; external calendar sync is a design-only
Connections (SA-031) concern. Cross-refs to both, and to the sibling
business-ops modules via the `@acct-<slug>` handle, are all soft
(present-if-mounted).

## See also

- [`BUILD_PLAN.md`](BUILD_PLAN.md) — the full spec + build plan.
- `content/schedule-rules.md` — the installed spine.
- `elements/module-notes/` — the shell precedent (spine + spec + seam +
  ledger, skills deferred) this module mirrors.
- `elements/module-scheduler/` — the generation-engine sibling; the boundary
  between records and computation is load-bearing.
- Canon `ELEMENT_CONTRACT.md` §7 — install policies. Canon Spec §6 — the
  `module` kind.
