# ZcodeGlobalQuotaClock v0.1 Architecture

> **Status:** FROZEN FOR STAGE 0/1 IMPLEMENTATION  
> **Project:** `Chengy257/ZcodeGlobalQuotaClock`  
> **Package / CLI:** `gqclock`

## 1. Purpose

ZcodeGlobalQuotaClock is a local account/provider-level quota-window maintainer.

Its responsibility is:

```text
observe provider quota
      ↓
identify real rolling-window boundary
      ↓
decide whether action is due
      ↓
optionally perform one authorized low-cost materialization
      ↓
re-observe provider quota
      ↓
confirm whether window identity advanced
```

It is deliberately independent from coding-task orchestration.

## 2. Responsibility boundary

### Owns

- provider/profile identity;
- quota observation and normalized snapshots;
- five-hour and weekly window semantics;
- provider-reported reset parsing;
- pure clock-target decisions;
- user-level durable clock state;
- materialization authorization;
- per-boundary attempt idempotency;
- low-cost materialization request;
- post-attempt confirmation;
- sanitized history;
- scheduler-facing single-shot `tick`.

### Does not own

- glm-conductor task IDs;
- Work Units or DAGs;
- repository ownership;
- validation/review;
- ZCode Workflow run IDs;
- task `waiting_quota`;
- bounded task-resume budgets;
- task wake bridges;
- code/project lifecycle state.

No field representing those concepts belongs in v0.1 schemas.

## 3. Runtime architecture

```text
                 ┌───────────────────┐
                 │  external scheduler│
                 │ Windows/systemd/...│
                 └─────────┬─────────┘
                           │
                     gqclock tick
                           │
                    acquire user lock
                           │
              ┌────────────▼────────────┐
              │   local state/config     │
              └────────────┬────────────┘
                           │
                 target already future?
                    │                │
                   yes              no
                    │                │
                  no-op       provider resolver
                                     │
                              normalized snapshot
                                     │
                               pure clock engine
                                     │
                          ┌──────────┴─────────┐
                          │                    │
                        not due               due
                          │                    │
                        no-op       authorized + safe?
                                               │
                                  ┌────────────┴───────────┐
                                  │                        │
                                 no                       yes
                                  │                        │
                              no active call       reserve boundary
                                                           │
                                                materialization call
                                                           │
                                                forced provider refresh
                                                           │
                                              confirm boundary advanced
                                                           │
                                                   write state/history
```

The scheduler does not understand quota semantics. It only invokes `tick`.

## 4. Target package layout

```text
src/gqclock/
├── __init__.py
├── cli.py
├── config.py
├── state.py
├── identity.py
├── clock.py
├── materialize.py
├── history.py
├── quota/
│   ├── __init__.py
│   ├── provider.py
│   ├── _http.py
│   ├── credentials.py
│   ├── parser.py
│   ├── resolver.py
│   ├── time_utils.py
│   ├── window_math.py
│   ├── zai.py
│   └── bigmodel.py
└── scheduler/
    ├── __init__.py
    ├── common.py
    ├── windows.py
    ├── systemd.py
    └── cron.py
```

## 5. State model

v0.1 state is user-level, not repository-level.

Default state root:

```text
~/.zcode-global-quota-clock/
├── config.json
├── state.json
├── history.jsonl
└── lock
```

The implementation may later adopt platform-native state directories, but Stage 0/1 should keep one deterministic path unless a concrete portability need justifies otherwise.

Conceptual schema:

```json
{
  "schema_version": 1,
  "profiles": {
    "default": {
      "provider": "zai",
      "provider_identity_hash": "...",
      "last_observed_at": "...",
      "last_snapshot": {
        "status": "AVAILABLE",
        "windows": []
      },
      "last_five_hour_reset_at": "...",
      "last_boundary_id": "...",
      "next_target_at": "...",
      "materialization": {
        "enabled": false,
        "boundary_id": null,
        "attempt_state": null,
        "attempted_at": null,
        "confirmed_at": null
      }
    }
  }
}
```

Never persist API keys, Authorization headers, raw provider responses, or full model request/response bodies.

## 6. Provider observation

Initial providers:

- Z.ai;
- BigModel.

Provider implementations expose one normalized snapshot contract; provider-specific transport details remain behind adapters.

Observation is passive. If provider data are unavailable or malformed, the normalized result is UNKNOWN / provider error and no active materialization is permitted.

## 7. Window semantics

### Five-hour boundary

The boundary is the provider-reported `reset_at`.

Never infer:

```text
now + 5 hours
```

as an equivalent.

### Weekly blocking

A blocking weekly window suppresses five-hour materialization.

If the weekly reset is parseable, the clock can park/recheck around that reset. If not parseable, it must retry observation later without inventing a time.

### Pure clock decisions

The pure clock module returns one of:

- `reset_target`
- `short_retry`
- `weekly_park`

It performs no I/O, state write, scheduler mutation, or model call.

## 8. Tick semantics

`gqclock tick --profile NAME` is single-shot and idempotent.

Recommended high-level order:

1. acquire exclusive user-level lock;
2. load config/state;
3. if a known target is safely in the future and no refresh is required, return no-op without network access;
4. otherwise force/perform provider observation as required;
5. derive current boundary/target;
6. if not due, persist observation/target and return;
7. if weekly blocked, suppress materialization;
8. if materialization disabled, return due-but-not-authorized;
9. reload/recheck boundary attempt state under the same lock;
10. durably reserve the boundary attempt;
11. issue at most one materialization request;
12. force-refresh provider quota;
13. confirm only if window identity/reset advanced;
14. persist result/history and exit.

A crash after durable reservation must not permit automatic resend for the same boundary.

## 9. Materialization state

Boundary attempt states may be represented as:

- `reserved`
- `attempted`
- `confirmed`
- `unconfirmed`

Scheduling/decision outputs may additionally describe:

- `not_due`
- `due_not_authorized`
- `suppressed_weekly`

Do not turn this into a generic workflow state machine.

## 10. Confirmation invariant

A model/API success response is not sufficient evidence.

Confirmation requires:

```text
post_attempt_boundary_id != pre_attempt_boundary_id
```

or an equivalent provider-observed advance in the five-hour window identity/reset.

Quota percentage change alone is also insufficient.

## 11. Concurrency model

One local process lock protects user-level state and the boundary reservation transaction.

Two simultaneous `tick` calls must result in at most one materialization request for a boundary.

No distributed locking is required in v0.1.

## 12. Scheduling boundary

Core correctness is scheduler-neutral.

Primary production scheduling:

- Windows Task Scheduler;
- systemd user timer;
- cron fallback.

Optional:

- ZCode Scheduled Task.

The project must never modify ZCode internal scheduler SQLite state.

## 13. CLI boundary

Planned stable v0.1 commands:

```text
gqclock status
gqclock plan
gqclock tick
gqclock doctor
gqclock history

gqclock enable-materialization
gqclock disable-materialization

gqclock schedule-plan
```

All machine-consumable commands should support single-object JSON output.

## 14. Failure philosophy

### Passive observation

Fail safe to no active action:

```text
provider unavailable/malformed
→ record sanitized diagnostic
→ no materialization
→ retry later
```

### Active materialization

Fail closed:

```text
authorization ambiguous
boundary ambiguous
state corrupt
reservation uncertain
→ do not call model
```

False negatives cost promptness; false positives consume real quota and alter provider state.

## 15. Version-0.1 non-goals

- cloud service;
- multi-user server;
- web UI;
- resident daemon;
- glm-conductor runtime dependency;
- coding-task resume;
- direct ZCode SQLite retiming;
- generic quota-budget optimization;
- auto-retrying ambiguous active calls.
