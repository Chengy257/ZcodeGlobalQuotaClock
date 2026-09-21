# Security and Active-Call Safety

## 1. Threat model

ZcodeGlobalQuotaClock handles provider credentials and may eventually make a real model request that consumes quota and changes provider-side rolling-window state.

Therefore passive observation and active materialization have intentionally different failure rules.

## 2. Credential invariants

The following must be mechanically tested:

1. API keys are never persisted by this project.
2. Authorization headers never appear in state/history/log output.
3. Exceptions must not include credential values.
4. Raw provider responses are not persisted by default.
5. Full materialization request/response bodies are not persisted.
6. Test fixtures are redacted synthetic/minimal snapshots.

Credentials may come from explicitly supported environment/config sources validated in Stage 0, but are memory-only after resolution.

## 3. Network invariants

Provider and materialization clients must enforce:

- HTTPS only;
- explicit host allowlist;
- bounded connect/read timeout;
- bounded response body;
- redirects disabled or fully revalidated against the allowlist;
- normalized safe error kinds;
- no forwarding credentials to an unapproved host.

Initial historical hosts to revalidate in Stage 0:

- `api.z.ai`
- `open.bigmodel.cn`

Do not freeze these merely because historical glm-conductor used them; Stage 0 must verify current provider behavior.

## 4. Observation vs active-call asymmetry

### Observation

If quota observation fails:

- return UNKNOWN / sanitized error;
- do not crash the scheduler repeatedly if a structured no-op can be returned;
- never treat missing data as authorization to materialize.

### Materialization

If any of these are uncertain:

- authorization;
- current boundary;
- reservation ownership;
- provider identity;
- active-call endpoint/model;
- whether an earlier ambiguous attempt may have reached the provider;

then do not send a new automatic call.

## 5. Explicit authorization

Default configuration:

```text
materialization.enabled = false
```

Enabling it must be an explicit user action tied to a named local profile.

A provider being exhausted does not itself authorize a model call.

## 6. One attempt per boundary

Automatic idempotency key:

```text
(profile/provider identity, five_hour boundary_id)
```

Once a boundary has a durable attempt reservation, no later automatic tick may send another request for that same boundary.

Ambiguous timeout is recorded as unconfirmed, not retried automatically.

## 7. Crash-safe ordering

Required transaction order:

1. lock;
2. re-read state;
3. re-evaluate due boundary;
4. durably reserve attempt;
5. send request;
6. force-refresh observation;
7. persist confirmed/unconfirmed result.

A process crash after step 4 must be safe: the next tick sees the reservation and does not resend.

## 8. History redaction

History may contain:

- timestamp;
- profile/provider;
- boundary id;
- decision;
- safe error kind;
- confirmation outcome.

History must not contain:

- secrets;
- Authorization header;
- raw URL with embedded credentials;
- raw body;
- user chat/code content;
- repository/task data.

## 9. ZCode boundary

Forbidden in v0.1:

- editing `tasks-index.sqlite`;
- modifying ZCode automation rows;
- discovering credentials by scraping undocumented internal databases;
- injecting commands into arbitrary coding sessions.

ZCode integration is scheduler-facing only.

## 10. CI safety

Normal CI must use fake providers/materializers and synthetic fixtures.

Real credential/network tests are opt-in, local/manual, and must never run automatically from pull requests or public CI.
