# ZcodeGlobalQuotaClock

**ZcodeGlobalQuotaClock** is a standalone, local quota-window maintainer for ZCode / GLM coding-plan accounts.

Its purpose is narrow:

> Observe the provider-reported rolling quota window, identify the real reset boundary, and—when explicitly authorized—perform at most one low-cost materialization request after the boundary so the next window becomes available promptly.

The project was extracted from the former Global Quota Clock subsystem in `glm-conductor` during the v2.4 native-workflow re-baseline. It is now an **independent project** with no runtime dependency on glm-conductor.

## Status

**v0.1 planning baseline — implementation ready.**

Implementation should start with **Stage 0 + Stage 1 only**. Active materialization is intentionally deferred until the live provider/materialization assumptions are revalidated and reviewed.

Authoritative implementation handoff:

- `docs/roadmap/V0_1_PROJECT_PLAN.md`
- `docs/roadmap/V0_1_IMPLEMENTATION_PLAN.md`
- `docs/ARCHITECTURE.md`

## What it does

Conceptually:

```text
provider/account identity
        ↓
quota-window observation
        ↓
provider-reported reset boundary
        ↓
clock decision
        ↓
authorized low-cost materialization
        ↓
forced post-observation
        ↓
confirm window identity advanced
        ↓
repeat
```

Primary planned CLI:

```text
gqclock status [--json]
gqclock plan [--json]
gqclock tick [--json]
gqclock doctor
gqclock history [--limit N]

gqclock enable-materialization <profile>
gqclock disable-materialization <profile>

gqclock schedule-plan --platform windows|systemd|cron|zcode
```

## What it does not do

ZcodeGlobalQuotaClock is **not**:

- a coding-task orchestrator;
- a ZCode Workflow manager;
- a glm-conductor task-resume service;
- a quota-budget allocator;
- a repository-aware agent runtime;
- a generic background-agent framework.

It does not know about Work Units, DAGs, repository ownership, reviewers, validation state, or Workflow run IDs.

## Key design rules

1. **Provider reset is authoritative.** Never replace a provider-reported five-hour reset with `now + 5h`.
2. **Observation is passive; materialization is active.** Active materialization is disabled by default.
3. **At most one automatic materialization attempt per boundary.**
4. **HTTP success is not proof of materialization.** The provider must report an advanced window identity/reset after the attempt.
5. **Ambiguous timeout is never auto-retried.**
6. **No credentials, Authorization headers, raw provider responses, or full model responses are persisted.**
7. **No writes to ZCode internal SQLite databases.**
8. **The core is scheduler-neutral.** Windows Task Scheduler / systemd / cron are primary production schedulers; ZCode Scheduled Tasks are optional convenience integration only.

## ZCode scheduling note

Current ZCode Scheduled Tasks are local and require the machine to be awake and the ZCode process to remain running. On Windows, closing the window normally hides ZCode to the system tray rather than quitting it, so scheduled work may continue; explicitly quitting ZCode stops it. ZCode custom repeats currently use hour/day/week/month/year units rather than minute-level intervals, so they are not the preferred v0.1 substrate for prompt five-hour boundary activation.

See `docs/SCHEDULING.md`.

## Repository layout target

```text
src/gqclock/
  cli.py
  config.py
  state.py
  identity.py
  clock.py
  materialize.py
  history.py
  quota/
  scheduler/

tests/
docs/
```

## Relationship to glm-conductor

The relationship is intentionally one-way at the design-history level only:

- this repository may adapt proven provider-observation behavior from glm-conductor history;
- this repository must not import glm-conductor at runtime;
- glm-conductor v2.4 must not require this project to be installed.

A future optional integration may consume `gqclock status --json` as a read-only diagnostic surface, but neither project owns the other's state.

## Documentation

- `docs/ARCHITECTURE.md` — current design truth
- `docs/SECURITY.md` — credential and active-call safety invariants
- `docs/SCHEDULING.md` — scheduler strategy and ZCode limitations
- `docs/PROVIDER_CONTRACT.md` — normalized quota/provider contract
- `docs/MIGRATION_FROM_GLM_CONDUCTOR.md` — what is reused, rewritten, and forbidden to migrate
- `docs/roadmap/V0_1_PROJECT_PLAN.md` — v0.1 scope and milestones
- `docs/roadmap/V0_1_IMPLEMENTATION_PLAN.md` — direct implementation handoff
