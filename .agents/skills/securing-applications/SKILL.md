---
name: securing-applications
description: "Secures application code on any stack: password hashing, injection, authorization, CSRF, CORS, XSS, secrets, security headers, security logging and audit, rate limiting, emailed links, file uploads, error responses that could leak internals, and dependency timeouts with circuit breakers. Use when writing or reviewing code that touches authentication, sign-in, sessions, tokens, JWT, password storage or hashing, CSRF, CORS, XSS, injection, authorization, secrets, audit trails, rate limiting, uploads or downloads, admin or permission changes, an OWASP-style security review, or when asked to explain a security claim."
---

# Security baseline

A project's own conventions override this skill.

## Core web rules

- Hash passwords with a slow memory-hard hash, Argon2id first, never a fast hash such as SHA-2 or MD5. OWASP minimum for Argon2id: 19 MiB memory, 2 iterations, 1 parallelism; use the project's numbers if set.
- Injection: parameterised queries only. Never build SQL, shell commands or HTML from input.
- Check authorization on the server for every object access, by tenant and owner, not only by role.
- Cookie sessions need CSRF defence: a token (synchronizer token) plus SameSite and an Origin check. SameSite alone is not enough. State changes never use GET.
- CORS: list allowed origins explicitly. Never reflect the request origin when credentials are allowed.
- XSS: encode output for its context (HTML, attribute, URL, script). Do not insert untrusted data as raw HTML.
- Secrets come from the environment or a secret store, never from source or images.
- Where the stack sets no defaults, send `X-Content-Type-Options: nosniff`, `Strict-Transport-Security`, a CSP or `frame-ancestors`, and `Referrer-Policy`.

## Secrets and personal data

- Never log or echo secrets, tokens, authorization codes or cookie values: not in code, tests, docs, commits or reports. Personal data appears only in the security log fields.
- A bad configuration value fails boot and names only the variable, never its value.
- Mutations are audited per the project's data-access rules (see `writing-database-code`, Tenancy).

## Logs and audit

- Security log lines carry only: source IP, user agent, keyed hash of the attempted identifier, internal user id. One helper writes every line.
- Audit by default, without asking, every user-affecting action: sign-in and security events, credential and status changes, link requests, settings writes. Audit rows hold no secrets, token values or client IP.
- Events with no known user go to application logs only (bots would grow the audit table without bound).
- Fields, hashing and routing table: `references/logging-and-audit.md`.

## Requests and errors

- An emailed link never changes state on a plain GET (mail scanners fetch links). Open a confirmation page with a button.
- Errors seen outside the server are minimal and generic. Internals never leak through an error path.
- Rate-limit keys hold no personal data: the user id when known, else a normalised keyed hash.
- Keep rate limits in the application, not a gateway: the app knows the user id and route tier, a gateway sees only IPs. Use a lower tier for unauthenticated routes. Count before any database read where cheap.
- Wrap every dependency call in a short timeout and a circuit breaker. This is resilience, kept here because a hung dependency is also a denial-of-service path. Default: after 2 to 3 failures, skip for 10 to 15 seconds, then probe (use the project's numbers if set).

## Refusals

- A new case that must be refused exactly like an existing one (status, body, timing) goes down the existing refused path unchanged, so callers cannot tell which accounts exist or their state. Branch only on what is logged. Tests assert both log lines and that the refusals are equal.
- A refactor that moves a validation inward can change which refusal a caller sees first. Keep the check at the same point: call a pure rule from the edge, and enforce it again in the core writer.
- Refuse actions that would orphan something critical, such as removing the last admin, and say why.

## Uploads and downloads

- Uploads go straight from the browser to object storage with a short-lived pre-signed URL; downloads use a short-lived link issued after an access check. The server never decodes untrusted images.
- Procedure: `references/uploads.md`.

## Explaining security

- Drop mitigations for threats that are not realistic: name the attacker and the step they take; if you cannot, the mitigation goes.

## References

- `references/logging-and-audit.md`: open when adding a log line, an audit event, or a rate-limit key.
- `references/uploads.md`: open when building or reviewing an upload or download flow.
