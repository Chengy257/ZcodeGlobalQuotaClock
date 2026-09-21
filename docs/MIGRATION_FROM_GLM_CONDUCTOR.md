# Migration from glm-conductor

## 1. Historical context

Global Quota Clock was previously implemented inside glm-conductor v2.3.

During the glm-conductor v2.4 Native Workflow re-baseline it was removed because account/provider-level window maintenance is semantically different from coding-task orchestration.

glm-conductor v2.4 now keeps only task-level quota observation and bounded Workflow resume.

This repository is the new authority for Global Quota Clock.

## 2. Source baseline

Historical handoff source:

- glm-conductor `docs/roadmap/GLOBAL_QUOTA_CLOCK_EXTRACTION_INVENTORY.md`
- glm-conductor v2.4 main line after merge commit `92509ae`

The inventory is evidence/history, not a runtime dependency.

## 3. Adapt/copy conceptually

The following behavior is valuable and may be adapted into this repository:

- quota provider abstraction;
- shared hardened HTTP transport;
- memory-only credential resolution;
- provider response parser;
- quota resolver;
- UTC/reset parsing helpers;
- window/reset math;
- Z.ai and BigModel observation adapters;
- pure Global Clock target decision table;
- Window Primer's materialization confirmation rules;
- Window Primer's single-attempt / ambiguous-timeout discipline.

Any copied code must be moved into this repository's namespace and tests. No runtime import from glm-conductor.

## 4. Rewrite for the standalone project

Rewrite rather than transplant:

- provider/profile identity;
- user-level config/state;
- history vocabulary;
- clock decision module API;
- materialization transaction;
- process lock;
- CLI;
- scheduler integrations;
- operational/security docs.

The standalone state must not inherit repository/task layout.

## 5. Do not migrate

Never migrate these glm-conductor concepts:

- task IDs;
- Work Units/DAGs;
- Workflow run IDs;
- task subscriptions;
- task wake bridge;
- task resume budgets;
- review/validation state;
- repository ownership;
- writer guard;
- task journal;
- activation transport.

Those belong to coding-task orchestration, not account-level quota maintenance.

## 6. Historical direct-SQLite scheduler adapter

The old clock wrote ZCode's internal scheduler SQLite field `next_run_at`.

Treat that only as historical evidence of a former workaround.

Do not copy:

- DB discovery;
- schema assumptions;
- retime SQL;
- compatibility probes whose sole purpose was supporting direct writes.

The standalone project uses scheduler-neutral periodic ticks.

## 7. Materialization lessons to preserve

The old Primer established several important invariants:

1. HTTP 200 is not materialization proof.
2. Percentage movement is not proof.
3. Provider-observed window identity/reset advancement is proof.
4. One automatic attempt per boundary.
5. Ambiguous timeout is not automatically resent.
6. Active call requires explicit authorization.
7. Unknown observation state cannot authorize an active call.

These principles are architectural, not legacy compatibility.

## 8. ZCode host observations to retain

Historical ZCode 3.14 testing showed scheduled turns are host-managed and not equivalent to clean new processes/sessions.

Current ZCode documentation should always override stale historical assumptions. In v0.1, ZCode scheduling remains optional; OS schedulers provide the preferred periodic trigger.

## 9. Independence test

A release of ZcodeGlobalQuotaClock passes the independence boundary only if:

- glm-conductor is not installed and all core features still work;
- no import/path reference to glm-conductor exists in runtime code;
- no glm-conductor task state is read;
- glm-conductor itself does not need this package to operate.
