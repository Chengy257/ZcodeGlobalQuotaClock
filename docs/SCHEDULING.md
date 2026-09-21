# Scheduling Strategy

## 1. Core principle

The clock core is scheduler-neutral.

Every scheduler only needs to invoke:

```text
gqclock tick --profile <name>
```

The command decides whether network observation or materialization is actually due.

## 2. Recommended cadence

For prompt five-hour boundary materialization, the recommended external trigger cadence is approximately **5–10 minutes**.

This does not mean every trigger must call the provider. `tick` should be able to return immediately from cached durable target state when the next target is clearly in the future.

This gives:

- prompt activation near the real reset;
- low steady-state network cost;
- no need for dynamic scheduler retiming.

## 3. Primary production schedulers

### Windows Task Scheduler

Preferred Windows production substrate.

Target properties:

- current user scope where practical;
- no administrator requirement unless necessary;
- fixed 5–10 minute trigger;
- invoke installed `gqclock` directly;
- sanitized output only;
- no embedded credentials in task arguments.

Stage 3 may generate a plan/script, but automatic installation requires explicit user action.

### systemd user timer

Preferred Linux option when systemd user services are available.

Use a user timer invoking the same single-shot CLI.

### cron

Fallback Linux/Unix option.

Use a simple periodic entry; all quota semantics remain inside `gqclock tick`.

## 4. Optional ZCode Scheduled Task integration

ZCode Scheduled Tasks are supported as a convenience mode, not the primary v0.1 correctness substrate.

Current ZCode documentation (verified 2026-09-21) states:

- tasks execute locally;
- the machine must be awake;
- nothing triggers if the ZCode process is actually closed;
- missed runs can be recorded as skipped rather than replayed;
- custom repeat units are hour/day/week/month/year;
- tasks can be bound to a session/project.

On Windows, current ZCode defaults to hiding the window to the system tray rather than quitting when the close button is clicked. In that case the process remains running and scheduled tasks can continue. Explicitly quitting ZCode stops that capability.

Consequences for this project:

- a ZCode task may periodically invoke `gqclock tick`;
- it must not be described as an offline service;
- hour-level custom recurrence is less precise than the recommended 5–10 minute OS scheduler cadence;
- this adapter is best-effort convenience, not the sole mechanism for prompt boundary activation.

## 5. No internal scheduler DB writes

The historical glm-conductor clock directly modified ZCode's internal `tasks-index.sqlite` `next_run_at` field to retime an automation.

That implementation must not be reintroduced.

v0.1 forbids:

- opening ZCode's internal scheduler DB for production writes;
- updating `next_run_at`;
- relying on undocumented table/column layouts.

## 6. Tick efficiency

A scheduler may run often, but `tick` should minimize work.

Suggested fast path:

```text
load state
↓
known next_target_at is safely in future?
├─ yes → no network, exit 0
└─ no  → observe provider and re-plan
```

Near a due boundary, provider observation is forced.

## 7. Overlap/concurrency

External schedulers may occasionally overlap triggers.

The user-level process lock must make overlap safe:

- only one tick performs active work;
- duplicate trigger exits/no-ops or waits briefly according to implementation policy;
- never permit duplicate materialization calls.

## 8. Missed triggers

If the machine is asleep or scheduler unavailable:

- do not replay multiple materialization attempts;
- the next successful tick observes the current provider state;
- if the boundary already advanced, no active call is needed;
- if it is still due and the boundary has no prior reservation, normal policy applies.

## 9. Future precision

If a future supported ZCode or OS API provides reliable one-shot scheduling at arbitrary timestamps, it may be added as a replaceable scheduler adapter.

The clock core should not need redesign.
