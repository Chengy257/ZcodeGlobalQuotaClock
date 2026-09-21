# Implementation Agent Instructions

This repository is currently in the **v0.1 implementation bootstrap** state.

## Authority order

Read in this order before editing:

1. `docs/ARCHITECTURE.md`
2. `docs/SECURITY.md`
3. `docs/PROVIDER_CONTRACT.md`
4. `docs/SCHEDULING.md`
5. `docs/roadmap/V0_1_IMPLEMENTATION_PLAN.md`
6. `docs/MIGRATION_FROM_GLM_CONDUCTOR.md`

If roadmap prose conflicts with `docs/ARCHITECTURE.md`, the architecture document wins unless the user explicitly approves a re-baseline.

## Current implementation scope

Start with **Stage 0 + Stage 1 only**.

Do not implement automatic materialization (Stage 2) until Stage 0 live assumptions and Stage 1 observation/core are reviewed.

## Hard rules

- No runtime import from `glm-conductor`.
- No ZCode internal SQLite writes.
- No credentials or Authorization values in repository files, test fixtures, logs, state, or history.
- No real provider/model calls in the normal unit/CI test suite.
- Materialization must default disabled.
- Unknown/malformed quota state must never trigger an active model call.
- Do not infer a five-hour boundary as `now + 5h`.
- Do not add task/Workflow/repository semantics from glm-conductor.
- Do not build a resident daemon in v0.1; the core entry is a single-shot idempotent `gqclock tick`.
- Do not auto-retry an ambiguous materialization timeout.

## Testing discipline

Use focused tests within each stage. Run one full regression at each stage exit, not after every edit.

Use deterministic fixtures, injected clocks, fake providers/materializers, and temporary state directories. Live probes belong in explicit opt-in scripts/results and must redact credentials/raw payloads.

## Stage handoff

Stage 0 + Stage 1 exit evidence should be written to:

`docs/reviews/V0_1_STAGE_0_1_REVIEW.md`

Do not proceed to Stage 2 automatically. The Stage 2 gate requires explicit review of the live provider/materialization assumptions.
