---
name: building-elysia-servers
description: "Conventions for JavaScript backends on Bun with Elysia: routes, services, TypeBox schemas, error handling, auth guards, bitmask RBAC, response shapes, rate limiting, logging, env config, Redis/Valkey, BullMQ queues and workers, and payment providers and webhooks. Use when building, modifying or reviewing Elysia backend code, adding a new module, or touching any of those topics."
---

# Elysia Server

## Runtime and tooling

Bun is the runtime and package manager. Do not use `npm`, `pnpm`, `yarn` or `node` for project commands.

```bash
bun --watch backend/server/src/index.js
bun install
bun test --cwd backend/server
bun add <package> --cwd backend/server
bun run lint
bun run fmt
```

- Tests use `bun:test`. Follow the `writing-tests` skill: unit tests on exported functions by default, hand-rolled fakes, no live DB, Redis/Valkey, queue or network. Ask the user before writing any other level.
- Test files live in the package's `test/` directory, mirroring `src/` (`test/modules/things/things.service.test.js`), never beside the source. Throwaway scripts go in `test/adhoc/` as plain `.js`, so `bun test` skips them.
- Keep logic in plain functions with I/O at the edges, so unit tests need no infrastructure.
- Prefer Bun native APIs where they fit.
- Use `oxlint` and `oxfmt`, print width 100. Leave the formatting of unchanged code alone.
- Write `.js` unless the project chooses TypeScript.
- Use Luxon `DateTime` for all date and time work, never native `Date`.

## References

Open a reference only when the task touches its topic.

- `references/http.md`: route, service and schema examples, validation split, error factories and the error handler, auth guards, bitmask RBAC, response shapes, app assembly.
- `references/workers.md`: BullMQ queues, worker processes, retries, the scheduler, the email worker.
- `references/redis.md`: only when the project has Redis/Valkey: key prefixes, cache, semaphores, rate-limit counters, emailed links.
- `references/payments.md`: only for payment providers: adapters, webhooks, reconciliation.
- `references/ops.md`: logging, env config, outbound calls (timeout and circuit breaker), operational safety.

Other skills own these topics. Do not restate them here.

- `writing-database-code`: queries, transactions, status codes, unique indexes, idempotency, ids, mappers.
- `securing-applications`: auth, audit, what a log line may carry, rate-limit keys, generic errors, emailed links, uploads.
- `writing-tests`: test levels and style.
- `structuring-backend-code`: layers, where shared code goes, one owner per rule.

## Architecture layers

```text
Routes (.routes.js) -> Services (.service.js) -> Shared queries (.queries.js) -> DB client
```

- Routes: HTTP only. Validate input, call a service, return the response.
- Services: business logic. Orchestrate queries, throw typed errors, enqueue jobs.
- Queries: one function per DB operation, no business logic.

## Package layout

Check which processes the project runs before placing code. Never carry paths over from another project.

- Server only: everything stays in the server package, including the DB client and `db/queries/`.
- Server plus workers (any second server-side process): move the DB client, query files and queue definitions into a server-side common package both import. One query function, two consumers.
- Constants and enums shared with the frontend get their own package. No server code (DB, queues, secrets) in it.
- Dependencies point one way: shared constants, then server common, then server and workers. Workers never import from the server package.

## Feature module

```text
modules/<feature>/
  <feature>.routes.js    # Elysia handlers
  <feature>.service.js   # business logic
  <feature>.schema.js    # TypeBox schemas
```

Database access goes in `<entity>.queries.js` files in the query layer for the package layout above, imported by the service.

## Routes, services, schemas

Code examples: `references/http.md`.

- Name every plugin: `new Elysia({ name: "module.routes" })`.
- Group routes with shared auth or permission needs under `.guard(...)`. Use a per-route `beforeHandle` when one route needs a permission.
- Return creations as `status(201, { entity })`. Destructure `user`, `body`, `query`, `params` and `status` from the handler context.
- Ids that are bigint travel as strings in JSON (see `writing-database-code`).
- Services are plain exported async functions, no classes. Private helpers stay unexported.
- Business validation and orchestration live in services, DB operations come from query files. Throw error factories (without `new`) for validation and business rules.
- Dispatch side effects through queues, never from routes.
- TypeBox covers structure only: shape, type, basic constraints. Apply it with `{ body }`, `{ query }` or `{ params }`. Never re-check in a service what a schema enforces.

## Quick reference

| Need | Pattern |
|------|---------|
| New module | routes + service + schema files |
| DB access | query function in the query layer (`writing-database-code`) |
| Auth required | `beforeHandle: [requireAuth()]` or `.guard(...)` |
| Permission | `beforeHandle: [requirePermission("DOMAIN", DOMAIN.FLAG)]` |
| Rate limit | `beforeHandle: [rateLimit("strict")]` or `rateLimit("moderate")` |
| Error | `throw NotFound("Not found", "ERROR_CODE")` |
| Side effect | `queue.add("jobName", payload)` from a service |
| Call to a dependency | short timeout plus circuit breaker (`references/ops.md`) |

## Common mistakes

- DB queries or business logic in route handlers.
- Business validation in schemas, or schema rules repeated in services.
- `new` with error factories.
- Queue jobs dispatched from routes.
- Native `Date` instead of Luxon.
- Tests that need live infrastructure, sit beside source, or assert nothing that can regress.
- Messages in errors that reveal whether an account, email or record exists.
