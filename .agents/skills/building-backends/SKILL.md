---
name: building-backends
description: 'Sets the rules for backend code in any language (Go, TypeScript, Rust, Java, Kotlin, Python): layers, one owner per rule, validation, typed errors with stable error codes, auth, permissions (bitmask RBAC), response shapes, package layout, queues and jobs, dependency injection and duplication. Points to the language skills `writing-go-code` and `building-elysia-servers`, and to `writing-database-code`, `securing-applications` and `writing-tests`. Use when writing, reviewing or refactoring a backend, server, API, service, handler, route or endpoint, or when validation, errors, error codes, permissions, RBAC, roles, jobs, queues, workers, package layout, duplication or a refactor comes up.'
---

# Building backends

These rules hold in every language. The stack skills add mechanics only: `writing-go-code`, `building-elysia-servers`. Other skills own their topics: `writing-database-code` (queries, transactions, idempotent writes), `securing-applications` (auth hardening, audit, generic errors, rate limits, uploads), `writing-tests`.

## References

- `references/errors-and-responses.md`: error types and codes, the global handler, database error mapping, validation split, auth flow, response shapes. Open when adding or changing an endpoint or an error.
- `references/permissions.md`: the bitmask permission model. Open when the app needs roles beyond "logged in".

## Layers

- Domain: types, invariants, validate and guard methods, policies (who may do what). No I/O.
- Service: one use case at a time. Calls domain methods, owns the transaction, throws typed errors, enqueues jobs. Never re-implements a domain rule.
- Adapters: HTTP or RPC handlers, the store or repository, external clients. They map input and output, call the service, and add transport detail (status code, hint). No business rules.
- Dependencies point inward: adapters import the service and domain; the domain imports neither.
- Dependencies are injected through constructors or parameters. No mutable global state. The entry point only reads config, wires and starts.

## One owner per rule

- The second time a predicate, policy, cap or check is needed, extract it into the layer that owns it (usually the domain) and call it everywhere. Never re-derive it inline. Copies drift: one handler answers 401 and three answer 403 for the same case.
- A domain validate or guard method is the path the code actually runs. A validator that only tests call is a defect: delete the hand-coded copies and call it.
- Each gate (deleted user, read-only credential, expired claim) is one function, checked at one point in the request. Check it before the lookup when the answer must not depend on whether the object exists.

## Validation, errors, auth

- Validate in two layers. The schema at the edge checks shape, types and basic constraints. The service checks business rules (existence, uniqueness, state, cross-field, ownership). Never re-check in a service what the schema enforces.
- Services throw typed errors that carry a stable `errorCode`. One global handler maps them to status codes and a generic body. Database errors map in that handler too. Details and the table: `references/errors-and-responses.md`.
- The client picks its message from `errorCode` (`building-frontends` relies on it). Every error carries one. A refusal the UI must word differently gets its own code.
- Handle each error once: return it or log it, not both. Log where it is finally mapped to a response or exit code.
- Resolve identity once per request, then guard per route or group (authenticated, then permission). The server is the security boundary; the client only hides UI.
- Roles beyond "logged in" use the bitmask model in `references/permissions.md`.

## Responses

One envelope per kind: single resource `{ entity }`, list `{ entities, total, page, limit }`, action `{ success, message }`, error `{ error, errorCode, requestId, ts }`. Creation returns 201. Ids that exceed the client's safe integer range travel as strings. Full contract: `references/errors-and-responses.md`.

## Package layout

Check which processes the project runs before placing code. Never carry paths over from another project.

- Server only: everything stays in the server package, including the store and queries.
- Server plus workers (any second server-side process): move the store, queries and queue definitions into a server-side common package both import. One query function, two consumers. Workers never import the server package.
- Constants and enums shared with the frontend get their own package. No server code (DB, queues, secrets) in it.
- Dependencies point one way: shared constants, then server common, then server and workers.

## Queues and jobs

- Side effects (email, notifications, payment follow-ups, anything slow or external) are enqueued from the service, never from a handler.
- Every job is safe to retry. Use idempotent writes (`writing-database-code`). Jobs with an external effect retry with backoff; idempotent sweeps and scheduled jobs run once and the next run is the retry.

## Where shared code goes

- Put shared code at the lowest layer that every user already imports, in a package named for what it holds (`money`, `pagination`), never a grab bag (`utils`, `common`, `helpers`).
- When packages of one layer import each other, move the shared piece into a helper package both import, never one importing the other. Enforce the direction with a guard (an import-rule test or a linter).
- Constants and enums used on both sides of a process boundary (client and server, CLI and API) live in one wire-types package both import. If one side may not import it, copy them and pin the copies with a test (see `writing-tests`).

## One shape per concept

- A filter or parameter set used by several routes, wire types and clients is one struct, embedded where needed, with one builder.
- Full and summary mappers for one entity: one builds on the other, so a new field cannot reach only one of them.
- A step repeated across handlers (emit an event, release a lock, create a link) is one function they call.
- A query fragment that must match an index exactly (a full-text expression) is one function.
- One parser per input kind (a CLI flag value, a query parameter). Every command or route that takes it uses it, so a value one accepts, all accept, and a missing required value is rejected the same way everywhere.

## Finding duplication

- Run a clone detector in the local gate (the stack skill names one). It catches copy-paste only.
- In every review, look for one rule written in different shapes: the same status, cap, permission or field list computed in two places. Fix it by extracting, never by syncing the copies.
