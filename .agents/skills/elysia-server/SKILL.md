---
name: elysia-server
version: 3
description: Use when building, modifying, or reviewing Elysia backend code — routes, services, error handling, validation, auth guards, RBAC permissions, response formatting, Redis/Valkey usage, queues, payment integration, or adding new modules
agents: [main_agent, general_purpose]
---

# Elysia Server

## Runtime & Tooling

Bun is the runtime and package manager. Do not use `npm`, `pnpm`, `yarn`, or `node` for project commands.

```bash
bun --watch backend/server/src/index.js
bun install
bun test --cwd backend/server
bun add <package> --cwd backend/server
bun run lint
bun run fmt
```

Key rules:

- Use `bun:test` for backend testing. Tests are unit-level — exported functions/methods — and by default run with zero infra: no live DB, Redis/Valkey, queues, or network. Infra-backed integration tests only when the user explicitly asks. (Frontend is the opposite: e2e only, see `frontend-conventions`.)
- Test files live under a per-package `test/` directory mirroring `src/` (e.g. `apps/api/test/modules/loans/loans.service.test.js`), never co-located beside source. Ad-hoc/throwaway scripts (smokes, one-off probes) go in `test/adhoc/` as plain `.js` — not `*.test.js`, so `bun test` skips them. `bun test` discovers tests anywhere, so no script/config change is needed.
- Every test must pin a behavior that can actually regress: a transformation, a branch, a boundary, a contract with consequences. Do not write field-passthrough assertions, `expect(...).not.toThrow()`, defaults the language already guarantees, or the same logic re-tested through trivially different inputs. Table-driven tests are fine only when each row pins a distinct real behavior. Test count is not a quality signal. When dispatching subagents to write tests, state this rule in the prompt and prune violations from their output.
- Prefer Bun native APIs where applicable.
- Use `oxlint` and `oxfmt` with print width 100.
- Prefer JavaScript `.js` for rapid prototyping unless the project explicitly chooses TypeScript.
- Use Luxon `DateTime` for all date/time work; never native JS `Date`.
- Respect existing formatting in unchanged code.

## Supporting References

- Use `http.md` for validation, error handling, auth guards, bitmask RBAC, response conventions, and app assembly.
- Use `workers.md` for BullMQ queues, background workers, schedulers, retries, and worker conventions.
- Use `redis.md` when the project uses Redis/Valkey — key namespacing, cache patterns, distributed semaphores, and concurrency claims. Skip it when there is no key-value store.
- Use `payments.md` when integrating payment providers — provider abstraction, webhooks, and reconciliation, framed as suggestions rather than rules.
- Use `ops.md` for logging, environment config, runtime reminders, and operational safety.
- For database queries, status conventions, constants, ID strategy, and mappers, use the `database-conventions` skill.

## Architecture Layers

```text
Routes (.routes.js) → Services (.service.js) → Shared queries (.queries.js) → DB client
```

- **Routes**: HTTP concerns only — validate input, call service, return response.
- **Services**: Business logic — orchestrate queries, throw typed errors, enqueue jobs.
- **Shared queries**: Pure DB operations in the query layer (see Package Layout); one function per operation, no business logic.

## Package Layout

Layout follows what the project actually runs — check which processes exist before placing code; never carry paths from one project into another:

- **Server only**: keep everything inside the server package; the DB client and `db/queries/` live there directly.
- **Server + workers** (any second server-side process): move the DB client, query files, queue definitions, and other shared server code into a server-side common package both processes import — one query function, two consumers, no duplication.
- Constants and enums shared with the frontend get their own FE+BE package; never put server code (DB, queues, secrets) in it.
- Dependency direction is one-way: shared constants package → server common package → server and workers. Workers never import from the server package.

## Feature Module Structure (3-File Pattern)

Every Elysia feature module follows this route/service/schema organization:

```text
modules/<feature>/
  <feature>.routes.js    # Elysia route handlers
  <feature>.service.js   # Business logic
  <feature>.schema.js    # TypeBox validation schemas
```

When a feature needs database access, put DB operations in query files (`<entity>.queries.js`) in the query layer for the project's package layout (see Package Layout), then import those query functions from the service. See the `database-conventions` skill for query, transaction, and mapping specifics.

## Routes

```js
import { Elysia } from "elysia";
import { requireAuth, requirePermission } from "../../middleware/rbac.middleware.js";
import { CreateThingBody } from "./things.schema.js";
import { createThing, listThings } from "./things.service.js";

export const thingsRoutes = new Elysia({ name: "things.routes" })
  .guard({ beforeHandle: [requireAuth()] }, (app) =>
    app
      .get("/api/things", async ({ user, query }) => {
        const result = await listThings(user.userId, query);
        return { things: result.items, total: result.total };
      })
      .post("/api/things", async ({ user, body, status }) => {
        const thing = await createThing(user.userId, body);
        return status(201, { thing });
      }, { body: CreateThingBody }),
  );
```

Route conventions:

- Use named Elysia plugins: `new Elysia({ name: "module.routes" })`.
- Group routes by shared auth/permission requirements with `.guard(...)`.
- Use per-route guards with `beforeHandle` when only one route needs a permission.
- Return creation responses as `status(201, { entity })`.
- Destructure `user`, `body`, `query`, `params`, and `status` from handler context.

## Services

```js
// query layer — location varies by package layout (see Package Layout)
import { insertThing } from "../../db/queries/things.queries.js";
import { BadRequest } from "../../lib/errors.js";

export async function createThing(userId, data) {
  if (data.name.length < 3) throw BadRequest("Name too short", "INVALID_NAME");
  return await insertThing({ ...data, userId });
}
```

Service conventions:

- Use plain exported async functions; no classes.
- Keep business validation and orchestration in services.
- Import DB operations from shared query files.
- Throw error factories for validation and business rules.
- Keep private helpers unexported at module scope.
- Dispatch side effects from services through queues, never directly from routes.

## Schemas

```js
import { t } from "elysia";

export const CreateThingBody = t.Object({
  name: t.String({ minLength: 1 }),
  description: t.Optional(t.String()),
  priority: t.Integer({ minimum: 1, maximum: 5 }),
  tags: t.Optional(t.Array(t.String(), { minItems: 1 })),
});
```

Schema conventions:

- Use TypeBox schemas for structural validation: shape, type, basic constraints.
- Keep business validation in services, not schemas.
- Do not duplicate schema-enforced rules in service code.
- Apply schemas declaratively on routes with `{ body }`, `{ query }`, or `{ params }`.

## Quick Reference

| Need | Pattern |
|------|---------|
| New module | Create routes + service + schema files |
| DB access | Add/import a query function from the project's query layer (see Package Layout + `database-conventions`) |
| Auth required | `beforeHandle: [requireAuth()]` or `.guard(...)` |
| Permission check | `beforeHandle: [requirePermission("DOMAIN", DOMAIN.FLAG)]` |
| Rate limit | `beforeHandle: [rateLimit("strict")]` or `rateLimit("moderate")` |
| Throw error | `throw NotFound("message", "ERROR_CODE")` |
| Async side effect | `queue.add("jobName", payload)` from service layer |
| Date/time | Luxon `DateTime` for all date/time work — never native JS `Date` |
| BigInt IDs | Use `{ mode: "string" }` and handle as strings |

## Common Mistakes

- Putting DB queries directly in route handlers.
- Putting business logic in route handlers.
- Putting business validation in schema files.
- Duplicating schema validation in service code.
- Using `new` with error factories.
- Dispatching queue jobs directly from routes.
- Using native JS `Date` instead of Luxon `DateTime`.
- Writing tests that require a live DB/Redis by default — unit tests run infra-free; integration tests only on request.
- Co-locating test files beside source in `src/` — they belong under the package's `test/` dir; throwaway scripts under `test/adhoc/` as plain `.js`.
- Padding test suites with passthrough/no-throw/duplicate-input tests — every test must pin a behavior that can regress.
