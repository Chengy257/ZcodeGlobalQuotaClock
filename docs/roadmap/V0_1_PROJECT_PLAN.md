# ZcodeGlobalQuotaClock v0.1 Project Plan

> **Status:** READY FOR IMPLEMENTATION  
> **Current execution gate:** Stage 0 + Stage 1  
> **Stage 2:** blocked pending Stage 0/1 review  
> **Stage 3:** blocked pending Stage 2 review

## 1. v0.1 objective

Deliver a standalone local utility that can reliably:

1. observe a supported GLM coding-plan account's rolling quota windows;
2. plan from provider-reported reset boundaries;
3. optionally perform one authorized low-cost materialization request per due five-hour boundary;
4. confirm materialization through post-request provider observation;
5. operate safely under repeated local scheduler invocation.

## 2. Release scope

v0.1 includes:

- Z.ai / BigModel quota observation;
- normalized provider contract;
- user-level config/state/history;
- pure reset/clock decision;
- passive `status/plan/doctor/history`;
- explicit materialization enable/disable;
- crash-safe one-attempt-per-boundary transaction;
- `tick`;
- Windows Task Scheduler guidance;
- systemd/cron guidance;
- optional ZCode Scheduled Task guidance;
- one real-boundary dogfood report;
- Linux + Windows CI.

## 3. Non-goals

- cloud deployment;
- web UI;
- multi-user service;
- resident daemon;
- coding-task resume;
- glm-conductor integration dependency;
- direct ZCode SQLite writes;
- provider-agnostic budget optimizer;
- aggressive retry of active calls.

## 4. Stage 0 — Bootstrap and live-assumption closure

### Purpose

Prevent stale glm-conductor-era assumptions from silently becoming the new implementation contract.

### Work

Freeze repository/package baseline and perform narrowly scoped live probes for:

- current quota endpoints;
- auth header shape;
- five-hour / weekly fields;
- provider-supplied reset values;
- forced refresh;
- candidate lowest-cost materialization model/API;
- pre/post observation behavior;
- current ZCode scheduling limitations.

### Outputs

- installable package skeleton;
- redacted provider fixtures;
- `docs/reviews/STAGE_0_LIVE_ASSUMPTIONS.md`;
- any necessary amendments to provider/security docs.

### Exit

Proceed only if provider observation and a candidate active-call mechanism are sufficiently understood.

Stage 0 live probes do **not** enable automatic materialization.

## 5. Stage 1 — Observation and pure clock core

### Work

Implement:

- `quota/provider.py`;
- hardened HTTP;
- credential resolver;
- parser/resolver;
- provider adapters;
- profile identity;
- state/history;
- time/window math;
- pure clock decision;
- `status`;
- `plan`;
- `doctor`;
- `history`.

### Tests

Use only deterministic fixtures/fakes in standard suite.

Test:

- parser variants;
- unknown additive fields;
- missing weekly window;
- malformed reset;
- multiple same-kind windows;
- provider error normalization;
- credential redaction;
- state atomicity;
- process lock;
- clock decision table;
- weekly suppression.

### Exit

Stage 0 + Stage 1 review must show:

- no secrets persisted;
- no active model call path exists yet;
- provider reset, not local +5h, drives decisions;
- passive CLI is usable independently of glm-conductor;
- tests/CI pass.

Write:

`docs/reviews/V0_1_STAGE_0_1_REVIEW.md`

Do not start Stage 2 automatically.

## 6. Stage 2 — Authorized materialization transaction

### Entry gate

Requires explicit approval after Stage 0/1 review.

### Work

Implement:

- materialization config;
- stable boundary identity;
- exclusive transaction lock;
- durable attempt reservation;
- isolated minimal materialization client;
- pre/post forced quota observation;
- confirmed/unconfirmed state;
- duplicate suppression;
- `enable-materialization`;
- `disable-materialization`;
- `tick`.

### Required failure injection

- concurrent ticks;
- crash after reservation;
- timeout after possible delivery;
- network failure;
- provider unavailable;
- weekly exhausted;
- reset already advanced;
- state corruption;
- materialization disabled.

### Exit

Mechanical proof:

- at most one automatic active call per boundary;
- ambiguous timeout never auto-retries;
- provider boundary advance is required for confirmation;
- unknown/malformed observation never triggers active call.

## 7. Stage 3 — Scheduling, dogfood, and v0.1 release

### Work

Add:

- Windows Task Scheduler plan/generator;
- systemd user timer plan;
- cron plan;
- optional ZCode Scheduled Task instructions;
- install/upgrade/uninstall docs;
- one real-boundary dogfood run.

### Dogfood evidence

Capture sanitized:

- pre-boundary snapshot;
- planned target;
- actual tick time;
- attempt reservation;
- active-call result class;
- forced post-observation;
- confirmation state;
- duplicate-tick result.

### Release gate

- one final full regression;
- package install/smoke;
- lint/static checks;
- credential leak scans;
- Linux/Windows CI;
- dogfood report;
- README operational instructions.

Release target: `v0.1.0`.

## 8. Testing cadence

Within a stage:

- focused tests during development;
- one stage-level full suite at exit.

Do not repeatedly run full regression after every work unit.

Real credentials/network calls remain opt-in and outside ordinary CI.

## 9. Versioning after v0.1

Potential later work, not part of v0.1:

- supported one-shot scheduler adapters;
- additional providers;
- richer health metrics;
- read-only local status socket/API;
- optional glm-conductor diagnostic integration;
- more precise scheduler wake strategy.
