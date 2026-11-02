# HTTP, validation, auth and errors

Rules (two-layer validation, error codes, auth flow, permission model, response shapes): `building-backends`. This file holds the Elysia code.

## Routes, services, schemas

```js
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

```js
// things.service.js
import { insertThing } from "../../db/queries/things.queries.js";
import { BadRequest } from "../../lib/errors.js";

export async function createThing(userId, data) {
  if (data.name.length < 3) throw BadRequest("Invalid request", "INVALID_NAME");
  return await insertThing({ ...data, userId });
}
```

The schema example is in the next section.

## Validation: TypeBox

Layer 1, the schema (route level): TypeBox checks shape, types and basic constraints. Failures reach the error handler as `VALIDATION`. Elysia's own default status for a failed validation is 422; this skill's handler maps it to 400 on purpose, so the API has one client-error status. Keep that mapping unless the project says otherwise.

```js
export const CreateThingBody = t.Object({
  name: t.String({ minLength: 1, maxLength: 200 }),
  email: t.String({ format: "email" }),
  priority: t.Integer({ minimum: 1, maximum: 5 }),
  tags: t.Optional(t.Array(t.String(), { minItems: 1 })),
  kind: t.Union([t.Literal("A"), t.Literal("B")]),
  note: t.Optional(t.Union([t.String(), t.Null()])),
  metadata: t.Optional(t.Record(t.String(), t.String())),
});
```

Layer 2, the service, is covered in `building-backends`.

## Errors

### Factories

```js
export class HttpError extends Error {
  constructor(status, message, errorCode, data) { ... }
}
export function BadRequest(message, errorCode, data) { return new HttpError(400, ...); }
// Unauthorized 401, Forbidden 403, NotFound 404, Conflict 409, TooManyRequests 429 likewise
```

Call them without `new`: `throw Conflict("Request conflicts", "THING_CONFLICT")`.

### Global handler

Register `.onError()` on the root app and `.error({ HttpError })` so the class is known. Handle:

- `VALIDATION` and `PARSE`: 400.
- `NOT_FOUND`: 404.
- `HttpError`: its own status.
- Postgres errors: `23505` and `23503` map to 409, `23502` to 500 (mapping rule: `building-backends`).
- Anything else: 500 with a generic body.

The body shape is in `building-backends`.

## Auth and guards

Default pattern: global resolve plus per-route guards. Change the mechanism only when the project already uses another.

### Global resolve plus per-route guards

Extract auth state once with `.resolve({ as: "global" }, ...)`. Each route decides whether it needs auth.

```js
export const authMiddleware = new Elysia({ name: "auth.middleware" })
  .resolve({ as: "global" }, async ({ cookie }) => {
    const user = await verifyToken(cookie.token?.value);
    return { user };
  });
```

### Guard factories

```js
export function requireAuth() {
  return ({ user }) => {
    if (!user) throw Unauthorized("Authentication required", "AUTH_REQUIRED");
  };
}

export function requirePermission(domain, flag) {
  return ({ user }) => {
    if (!user) throw Unauthorized("Authentication required", "AUTH_REQUIRED");
    if (!canDo(user.roleId, domain, flag)) throw Forbidden("Forbidden", "FORBIDDEN");
  };
}
```

```js
.get("/api/admin/users", handler, { beforeHandle: [requirePermission("USR", USR.USERS_VIEW)] })

.guard({ beforeHandle: [requireAuth()] }, (app) =>
  app.get("/api/me", handler).patch("/api/me", handler),
)
```

### Permission check

Model and rules: `building-backends` (`references/permissions.md`).

```js
export const CON = Object.freeze({
  CONTENT_VIEW: 1 << 0,
  CONTENT_CREATE: 1 << 1,
  CONTENT_MANAGE: 1 << 2,
});

export function canDo(roleId, domain, flag) {
  const perms = ROLE_PERMISSIONS.get(roleId);
  return perms ? (perms[domain] & flag) !== 0 : false;
}
```

### Global context

`.derive({ as: "global" }, ...)` injects per-request context such as `requestId`.

## Response shapes

Contract: `building-backends`. Creation uses `status(201, { entity: data })`.

## App assembly

```js
const app = new Elysia()
  .error({ HttpError })
  .use(cors(...))
  .use(requestLoggerMiddleware)
  .use(errorMiddleware)
  .use(authMiddleware)
  .use(rateLimitMiddleware)
  .use(moduleARoutes)
  .use(moduleBRoutes)
  .listen(port);
```

Order matters: logging and request id, error handler, auth, rate limiting, route modules.
