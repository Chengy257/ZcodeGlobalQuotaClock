# ZcodeGlobalQuotaClock v0.1 Detailed Implementation Plan

> **Audience:** local Codex / coding agent  
> **Authority:** `docs/ARCHITECTURE.md`  
> **Immediate implementation scope:** **Stage 0 + Stage 1 only**

## 0. Global implementation rules

- Use Python with a `src/` package layout.
- Console command: `gqclock`.
- Keep dependencies minimal; stdlib-first is preferred for the initial core.
- Do not import glm-conductor.
- Do not write ZCode internal SQLite.
- Do not implement active materialization before Stage 0/1 review.
- Do not use real credentials/network in standard tests.
- Keep state under a temporary override in tests; never touch the user's real state directory.
- Use focused tests during work and one complete suite at each stage exit.

## 1. Bootstrap work unit B0

Create:

```text
pyproject.toml
src/gqclock/__init__.py
src/gqclock/cli.py
src/gqclock/config.py
src/gqclock/state.py
src/gqclock/identity.py
src/gqclock/clock.py
src/gqclock/history.py
src/gqclock/quota/*
src/gqclock/scheduler/*
tests/
```

Minimum package metadata:

- project name: `zcode-global-quota-clock` or another PyPI-safe equivalent;
- console script: `gqclock = gqclock.cli:main`;
- version: `0.1.0.dev0`;
- supported Python baseline must be explicitly chosen and tested.

Do not add a daemon/service entry point.

### B0 tests

- package imports;
- `gqclock --help`;
- state-root override fixture;
- no runtime glm-conductor imports.

## 2. Stage 0 work unit S0-A — provider live probe harness

Create an opt-in probe script/module separate from normal CLI/CI.

It must:

- require explicit profile/credential availability;
- print only redacted normalized facts;
- support forced refresh;
- never persist raw responses;
- never print Authorization values.

For each initial provider, record:

- host/path;
- auth header form;
- response field names required for parsing;
- five-hour window identification;
- weekly window identification or absence;
- reset timestamp form;
- force-refresh behavior.

Write only sanitized evidence to:

`docs/reviews/STAGE_0_LIVE_ASSUMPTIONS.md`

and synthetic/redacted fixtures to `tests/fixtures/`.

## 3. Stage 0 work unit S0-B — materialization probe

Implement a **manual probe only**, not production auto-materialization.

It should verify:

- current eligible endpoint;
- current lowest-cost suitable model;
- minimum request shape;
- auth form;
- bounded timeout;
- whether a post-call forced quota observation can see an advanced boundary when run at a real boundary.

Safety:

- explicit command-line acknowledgement/flag;
- one invocation only;
- no loop/retry;
- no raw response persistence;
- sanitized result class only.

If a real boundary is not available during Stage 0, record the endpoint/model probe separately and mark boundary-advance confirmation as pending for Stage 2 dogfood.

## 4. Stage 0 work unit S0-C — scheduling fact verification

Update `docs/SCHEDULING.md` only if live/current evidence differs.

Anchor current ZCode behavior:

- machine awake required;
- ZCode process running required;
- Windows close-to-tray is not equivalent to quit;
- missed triggers may be skipped;
- custom repeat units are not minute-level.

No host database inspection/writes.

## 5. Stage 0 exit gate

Required:

- package skeleton imports;
- current provider assumptions documented;
- sanitized fixtures exist;
- no secret leakage;
- candidate materialization call understood;
- no automatic active-call code exists.

Commit a Stage 0 checkpoint before Stage 1.

## 6. Stage 1 work unit S1-A — provider/security core

Implement `quota/provider.py`, `quota/_http.py`, `quota/credentials.py`.

Required:

- `QuotaProviderError(kind, message)`;
- safe error kinds;
- HTTPS allowlist;
- redirects rejected/revalidated;
- timeout;
- max response size;
- memory-only credential values;
- no raw-response persistence.

Tests must inject recognizable fake secrets and assert they never appear in errors/state/history.

## 7. Stage 1 work unit S1-B — parser and resolver

Implement normalized snapshot contract from `docs/PROVIDER_CONTRACT.md`.

Parser requirements:

- tolerate additive unknown fields;
- support missing weekly window;
- reject/mark malformed core shapes safely;
- normalize reset timestamps;
- distinguish `five_hour` and `weekly`;
- produce clock-facing four-state status.

Provider adapters remain thin.

## 8. Stage 1 work unit S1-C — profile identity

Implement profile-based identity.

Preferred state key input:

```text
provider + local profile name + endpoint family
```

Optionally include a validated non-secret provider account identifier.

Do not derive identity solely from API key hash.

## 9. Stage 1 work unit S1-D — atomic state/history/lock

Implement:

- deterministic state root;
- atomic JSON state write;
- append-only sanitized JSONL history;
- exclusive process lock;
- corruption handling that never authorizes active behavior.

Expected history events in Stage 1:

- `observed`
- `target_planned`
- `provider_error`

Do not add materialization events until Stage 2.

## 10. Stage 1 work unit S1-E — pure clock

Implement pure decision logic.

Inputs:

- normalized snapshot;
- injected/current time;
- optional previous boundary;
- grace;
- retry delay.

Outputs:

```text
reset_target
short_retry
weekly_park
```

Required cases:

1. no usable snapshot → short_retry;
2. blocking weekly → weekly_park if reset parseable, otherwise short_retry;
3. five-hour missing → short_retry;
4. five-hour reset malformed → short_retry;
5. reset elapsed but provider has not advanced → short_retry;
6. future reset → reset_target at reset + grace.

No I/O in `clock.py`.

## 11. Stage 1 work unit S1-F — passive CLI

Implement:

### `gqclock status [--profile NAME] [--json]`

- observe current quota;
- show windows/status;
- update safe observation state/history.

### `gqclock plan [--profile NAME] [--json]`

- force/current observation as specified;
- run pure clock;
- persist next target;
- never active-call a model.

### `gqclock doctor [--profile NAME] [--json]`

Check:

- config;
- credentials presence without printing value;
- provider selection;
- state readability;
- lock ability;
- optional live observation only when explicitly requested.

### `gqclock history [--profile NAME] [--limit N] [--json]`

Read sanitized history only.

No `tick` active behavior yet; a Stage 1 `tick` placeholder may return "materialization not implemented" only if needed for packaging, but it must never issue a model call.

## 12. Stage 1 tests

Add deterministic tests for:

- provider safe errors;
- HTTP host/HTTPS/redirect/size limits;
- credential redaction;
- parser variants;
- missing weekly;
- malformed reset;
- duplicate windows;
- identity stability;
- state atomicity;
- process lock;
- corrupt state;
- history redaction;
- full clock decision table;
- CLI JSON shape;
- independence from glm-conductor.

## 13. Stage 0/1 integration smoke

Using fake provider fixtures:

```text
configure temp profile
→ gqclock status --json
→ gqclock plan --json
→ inspect temp state/history
→ repeat plan
→ verify deterministic/no secret leakage
```

Then optionally run the explicit live observation probe separately.

## 14. Stage 0/1 review artifact

Create:

`docs/reviews/V0_1_STAGE_0_1_REVIEW.md`

Required sections:

- implementation commit;
- Python/platform matrix;
- provider probe evidence;
- materialization probe status;
- current ZCode scheduling facts;
- package/CLI implemented surface;
- tests;
- security checks;
- unresolved assumptions;
- decision:
  - `STAGE_2_READY`
  - `STAGE_2_NOT_READY`

Do not start Stage 2 unless this review says `STAGE_2_READY`.

---

# Stage 2 — specification after review

## 15. S2-A materialization config/CLI

Only after approval.

Add:

- `enable-materialization PROFILE`;
- `disable-materialization PROFILE`;
- explicit config state.

Default remains disabled.

## 16. S2-B boundary identity and reservation

Define deterministic boundary id from profile/provider identity + provider-reported five-hour reset.

Implement under process lock:

```text
observe
→ due?
→ reload state
→ reserve boundary durably
→ active call
→ force refresh
→ confirm/unconfirmed
```

No second automatic call once reserved.

## 17. S2-C materialization client

Use the Stage 0-validated endpoint/model only.

Requirements:

- fixed innocuous minimal prompt;
- minimal output;
- bounded timeout/body;
- sanitized exceptions;
- no code/chat/repository context.

## 18. S2-D confirmation

Confirmed iff post-observation shows the five-hour boundary advanced.

Not confirmed by:

- HTTP 200;
- text output;
- percentage movement alone.

## 19. S2-E production `tick`

Implement fast path + due path.

Required outcomes should be machine-readable and distinguish:

- not_due;
- observation_unknown;
- weekly_suppressed;
- due_not_authorized;
- attempt_reserved;
- attempted_unconfirmed;
- confirmed;
- already_attempted.

Do not return an error code that encourages external blind retry after an ambiguous active request.

## 20. Stage 2 failure-injection gate

Prove:

- concurrent ticks produce ≤1 active call;
- crash after reservation → later tick does not resend;
- timeout → no auto retry;
- malformed/unknown observation → no active call;
- weekly exhausted → no five-hour active call;
- boundary already advanced → no active call;
- materialization disabled → no active call.

Write Stage 2 review before Stage 3.

---

# Stage 3 — scheduling and release specification

## 21. S3-A scheduler plans

Implement/generate instructions for:

- Windows Task Scheduler (primary Windows);
- systemd user timer;
- cron;
- optional ZCode Scheduled Task.

Prefer fixed periodic triggers; keep quota semantics in `tick`.

## 22. S3-B real-boundary dogfood

Run at least one controlled five-hour boundary with explicit authorization.

Record sanitized evidence:

- old boundary;
- target;
- actual tick;
- reservation;
- active request result;
- forced post-observation;
- confirmed/unconfirmed;
- repeated tick no-duplicate behavior.

## 23. S3-C release closeout

Run once after all fixes:

- full unit/integration suite;
- lint/static;
- packaging install smoke;
- Linux/Windows CI;
- credential leak scans.

Release `v0.1.0` only if dogfood and security gates pass.

## 24. Commit discipline

Prefer substantial checkpoint commits:

1. bootstrap + Stage 0 probe framework;
2. provider/parser/security core;
3. state/clock/passive CLI;
4. Stage 0/1 review;
5. Stage 2 materialization transaction;
6. Stage 3 scheduling/release.

Avoid one commit per tiny test unless required for debugging.

## 25. Immediate handoff prompt

Use this to begin implementation:

> Implement **Stage 0 + Stage 1 only** from `docs/roadmap/V0_1_IMPLEMENTATION_PLAN.md`. Treat `docs/ARCHITECTURE.md` and `docs/SECURITY.md` as hard contracts. Revalidate live provider/materialization assumptions through explicit redacted probes; do not implement automatic materialization. Do not import glm-conductor or write ZCode internal SQLite. Use deterministic fake-provider tests for CI, focused tests during work, and produce `docs/reviews/V0_1_STAGE_0_1_REVIEW.md` with a final `STAGE_2_READY` or `STAGE_2_NOT_READY` verdict.
