---
name: security-baseline
description: Use when writing or reviewing code that touches authentication, sign-in, sessions, tokens, secrets, logging, audit trails, rate limiting, emailed links, file uploads or downloads, error responses, dependency calls (timeouts, circuit breakers), admin or permission changes, or when asked for a security review or to explain a security claim.
---

# Security baseline

Defaults for any stack. A project's own rules override them where they conflict.

## Secrets and personal data

- Never log or echo secrets, tokens, authorization codes or cookie values: not in code, tests, docs, commits or reports. Personal data appears only in the security log fields listed below.
- A bad configuration value fails boot and names only the variable, never its value.
- Mutations are audited per the project's data-access rules.

## Logs and audit

- Security log lines may carry only: source IP, user agent, a keyed hash of the attempted identifier, the internal user id.
- Never log the raw identifier, passwords, tokens, cookie values, authorization codes or any other personal data.
- One helper writes every security log line, so the allowed fields live in one place.
- Audit every sensible user action that affects the user, by default, without asking: sign-in and security events, credential and status changes, link requests, settings writes.
- Audit rows hold no secrets or token values and no client IP.
- Events with no known user (sign-in with an unknown email) go to application logs only. Bots could grow the audit table without bound.
- Examples and field lists: `references/logging-and-audit.md`.

## Requests and errors

- An emailed link never changes state on a plain GET. Mail scanners and previewers fetch links. Open a page that asks for confirmation, with a button as the fallback.
- Errors seen outside the server are minimal and generic. Internals never leak through an error path.
- Rate-limit keys hold no personal data: the user id when known, else a normalised keyed hash.
- Keep rate limits in the application, with a lower tier for unauthenticated routes. Count before any database read where that is cheap.
- Wrap every call to a dependency in a short timeout and a circuit breaker. Default: after a couple of failures skip the dependency for 10 to 15 seconds, then probe. Use the project's numbers if it sets them. One hung dependency must not slow the whole API.

## Refusals

- A new case that must be refused exactly like an existing one (status, body and timing) goes down the existing refused path unchanged. Branch only on what is logged. Tests assert both log lines and that the two refusals are equal.
- A refactor that moves a validation inward can change which refusal a caller sees first. Keep the check at the same point: call a pure rule from the edge, and enforce it again in the core writer.
- Refuse actions that would orphan something critical, such as removing the last admin, and say why.

## Uploads and downloads

- Uploads go straight from the browser to object storage with a short-lived pre-signed URL. The server checks access, size and type, and never decodes untrusted images.
- Downloads use a short-lived pre-signed link, issued after an access check.
- Procedure: `references/uploads.md`.

## Explaining security

- Explain each security claim in plain words with the concrete failure or attack.
- Drop mitigations for threats that are not realistic. Name the attacker and the step they take; if you cannot, the mitigation goes.

## References

- `references/logging-and-audit.md`: open when adding a log line, an audit event, or a rate-limit key.
- `references/uploads.md`: open when building or reviewing an upload or download flow.
