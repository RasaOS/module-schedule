# CLAUDE.md — `rasa.module.schedule`

> **Who you are (SA-025).** `rasa.module.schedule` — the RasaOS module for scheduling. Substrate: **RasaOS**; role: **module**. On install `bin/init` renders this into `.claude/rasa-identity.md`; `/whoami` composes the full identity with the project's deployment layer.


Per-repo working contract for Claude sessions opened inside this folder.
Extends `~/.claude/CLAUDE.md` and the workspace `~/rAI/rasa-os/CLAUDE.md`
(the `rasa.tenant.rasaos` tenant's contract); does not override them.

## What you are when you're in this folder

You are working on **`rasa.module.schedule`** — a `module`-kind, business-ops
Element: the **bookings-as-records** module (the calendar face), one of six
business-ops shells (`leads`, `schedule`, `scheduler`, `contracts`,
`invoices`, `field-log`). It owns ONE record — the **Booking** — in a
project-owned git-versioned markdown ledger (`schedule/SCHEDULE.md`), with the
lifecycle `tentative → confirmed → done | cancelled`.

A `module` (canon Spec §6) is "a focused capability that extends a domain or
orchestrator, mountable into one or more parents." Parents pull it via their
own `requires.elements[]`.

## The load-bearing ideas

**1. Bookings-as-records — deliberately lightweight.** A booking is a row in
a markdown ledger, not a scheduling computation. The record layer stays dumb
and durable on purpose. If you find yourself writing a solver here — **stop**;
that's the engine's job (see idea 3).

**2. The account spine.** Every Booking references its customer by the shared
`@acct-<slug>` handle — the same handle `leads`, `contracts`, `invoices`, and
`field-log` use, so records join across the modules by shared key. The handle
is **optional** here (internal bookings omit it). This module never owns or
mutates the account's own record; the account hub is the missing primitive
tracked as canon task **SA-032**. Do not seed a competing accounts registry.

**3. The critical boundary — records, NOT the generation engine.**
Availability solving, recurrence expansion, conflict resolution → that is
`rasa.module.scheduler` (soft sibling, present-if-mounted). This module holds
the resulting bookings. External calendar sync (Google/Outlook/ICS) is a
**design-only** Connections (canon task **SA-031**) reference — NOT
implemented here. Reminder delivery is out of scope; a booking records
reminder *intent* only.

**4. The schedule-gate seam.** The one per-project thing — working hours,
timezone, booking types, reminder policy — is the project-owned
`.claude/schedule-gate.md` (seeded skip-if-exists). `content/` references it;
it hardcodes none of it.

## Skills are deferred (extract-after-proof)

**v0.1.0 = spine + spec + seam + ledger template. The `/book`, `/schedule`,
`/remind` skills are NOT built yet** (see `content/BUILD_PLAN.md` M-1..M-3).
This is deliberate: the skill build is gated on a real consumer declaring the
module in its `requires.elements[]` (the `module.tasks` extract-after-proof
precedent). Do not author the skills without the user driving that decision.

## Source of truth

- **`~/rAI/rasa-os/canon/`** — authoritative. Spec §6 defines the `module`
  kind; ELEMENT_CONTRACT.md §7 the install policies.
- **`content/schedule-rules.md`** — the installed spine (the record + the
  discipline).
- **`content/BUILD_PLAN.md`** — the spec + the skill contracts + the gate.
- **`rasa.json`** — the formal declaration + install manifest.
- **`~/rAI/rasa-os/elements/REGISTRY.md`** — the live workspace snapshot.

## Don'ts

- **You are NOT the template.** If this contract ever describes
  `domain-core` (or any other Element), the template-CLAUDE.md drift class
  is back — flag it.
- **Don't build the generation engine here** (availability, recurrence,
  conflict resolution). Soft-reference `module.scheduler`; never harden it
  into `requires.elements[]`.
- **Don't implement external calendar sync.** SA-031 / Connections is
  design-only; the local ledger is the source of truth.
- **Don't seed an accounts registry.** The `@acct-<slug>` handle is the join;
  the account hub is SA-032.
- **Don't author the skills ahead of the gate** without the user driving.
- **Don't `bin/init` this Element into itself.** `content/` is the source.
- **Don't push from the Cowork sandbox.** Local commit + tag only; the user
  pushes from their machine (workspace rule).

## How a version bump works

- **Patch** — wording/template fix. **Minor (→0.2.0)** — the skills land
  (once the gate opens), a new seed file, a new capability. **Major
  (→1.0.0)** — seam format + install shape locked after a real vertical runs
  the record layer unchanged.

Each bump: edit `VERSION` + `rasa.json#version`, write a CHANGELOG entry,
run `bin/check-manifest`, commit + tag `v<version>`. Update
`~/rAI/rasa-os/elements/REGISTRY.md` + `~/rAI/rasa-os/elements/CHANGELOG.md`
(track #2).
