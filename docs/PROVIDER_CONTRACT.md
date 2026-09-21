# Provider and Quota Contract

## 1. Purpose

All provider-specific APIs must normalize into one small quota snapshot consumed by the clock core.

The clock core must not depend on provider-native response structure.

## 2. Provider interface

Conceptual Python contract:

```python
class QuotaProvider:
    def fetch(self, force: bool = False) -> dict:
        ...
```

Failures should raise a normalized provider error rather than leak raw transport exceptions.

Recommended safe error kinds:

- `auth`
- `unavailable`
- `malformed`
- `network`
- `unknown`

## 3. Normalized snapshot

Target shape:

```json
{
  "provider": "zai",
  "profile": "default",
  "fetched_at": "2026-09-21T12:00:00Z",
  "status": "AVAILABLE",
  "windows": [
    {
      "kind": "five_hour",
      "status": "AVAILABLE",
      "used_percent": 20,
      "remaining_percent": 80,
      "reset_at": "2026-09-21T15:00:00Z"
    },
    {
      "kind": "weekly",
      "status": "AVAILABLE",
      "used_percent": 40,
      "remaining_percent": 60,
      "reset_at": "2026-09-25T00:00:00Z"
    }
  ]
}
```

Optional provider metadata may be retained only when it is non-secret and useful for diagnostics.

## 4. Status vocabulary

Clock-facing observation status:

- `AVAILABLE`
- `PRESSURE`
- `EXHAUSTED`
- `UNKNOWN`

Window status should use the same or a strictly documented subset.

UNKNOWN must never be interpreted as permission to perform an active call.

## 5. Window kinds

v0.1 understands at minimum:

- `five_hour`
- `weekly`

Unknown additive provider windows should not break parsing. Preserve them only if needed for diagnostics; clock decisions ignore them unless later specified.

## 6. Reset semantics

`reset_at` is provider-supplied and authoritative.

Rules:

- parse as timezone-aware UTC;
- malformed reset → unknown/unparseable for that decision;
- never synthesize `now + 5h`;
- when multiple same-kind windows exist unexpectedly, the clock implementation must apply one deterministic documented rule and test it.

Historical Global Clock behavior used the latest parseable relevant reset for next-target decisions. Stage 0/1 should preserve that unless current provider evidence gives a reason to change it.

## 7. Weekly suppression

If a weekly window is EXHAUSTED/blocking, five-hour materialization is suppressed.

The next clock decision may park around the weekly reset, but only when that reset is parseable.

## 8. Provider identity

Do not identify an account solely by hashing an API key.

Preferred identity inputs:

1. non-secret provider account/profile identifier when available;
2. otherwise explicit local profile name + provider + endpoint family.

The resulting `provider_identity_hash` is a local state key, not an authentication artifact.

## 9. Caching

A small in-process TTL cache is allowed for user-facing repeated status calls.

`force=True` must bypass it for:

- live Stage 0 probes;
- pre/post materialization confirmation;
- explicit refresh diagnostics.

Persistent raw response caching is forbidden.

## 10. Initial adapters

Stage 0 should independently revalidate the historical Z.ai and BigModel quota endpoints and auth shape before freezing implementation fixtures.

Do not assume historical endpoint paths are permanent.
