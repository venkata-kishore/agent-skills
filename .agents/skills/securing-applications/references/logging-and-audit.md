# Logging and audit

## Where an event goes

| Event | Destination |
|---|---|
| Action by a known user that affects that user | audit row |
| Failed sign-in for a known user | audit row, plus security log line |
| Sign-in attempt for an unknown identifier | security log line only |
| Refusal of a deactivated account | security log line only, same response as any other refusal |

## Security log line

- Allowed: source IP, user agent, keyed hash of the attempted identifier, internal user id.
- Never: raw identifier, password, token, cookie value, authorization code, other personal data.
- Hash with a keyed function (HMAC) over the normalised identifier (trimmed, case-folded), so the same input always gives the same value and a leaked log cannot be reversed by guessing.
- Route every line through one helper. A call site cannot add a field the helper does not take.

## Audit row

- Actor, action, target, time, and a result.
- No secrets, no token values, no client IP.
- Audit by default for each user-affecting action. Do not ask whether to audit.
- Mutations are audited per the project's data-access rules; do not invent a second mechanism.

## Rate-limit key

- Known user: the user id.
- Unknown user: a normalised keyed hash of the identifier, never the raw value.
- Unauthenticated routes get a lower limit than authenticated ones.
- Count before the first database read where that is cheap.
