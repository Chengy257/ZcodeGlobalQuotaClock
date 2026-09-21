# Documentation Index

## Current truth

- [ARCHITECTURE.md](ARCHITECTURE.md) — v0.1 architecture and responsibility boundary
- [SECURITY.md](SECURITY.md) — credentials, network, state, and materialization safety invariants
- [PROVIDER_CONTRACT.md](PROVIDER_CONTRACT.md) — normalized quota provider contract
- [SCHEDULING.md](SCHEDULING.md) — scheduler-neutral execution model and ZCode/OS integration
- [MIGRATION_FROM_GLM_CONDUCTOR.md](MIGRATION_FROM_GLM_CONDUCTOR.md) — extraction disposition from glm-conductor

## Implementation roadmap

- [V0_1_PROJECT_PLAN.md](roadmap/V0_1_PROJECT_PLAN.md) — product scope, stages, release definition
- [V0_1_IMPLEMENTATION_PLAN.md](roadmap/V0_1_IMPLEMENTATION_PLAN.md) — implementation-grade Stage 0–3 handoff

## Review outputs

Implementation reviews should be written under `docs/reviews/`.

First expected review:

- `docs/reviews/V0_1_STAGE_0_1_REVIEW.md`

## Historical source

The historical Global Quota Clock lived in glm-conductor v2.3 and was removed from glm-conductor runtime in v2.4. The extraction inventory remains in the glm-conductor repository as historical evidence; this repository is now the authority for all new Global Quota Clock design and implementation.
