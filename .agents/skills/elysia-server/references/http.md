# HTTP, validation, auth and errors

## Validation: two layers

Layer 1, the schema (route level): TypeBox checks shape, types and basic constraints. Failures reach the error handler as `VALIDATION` and answer 400.

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

Layer 2, the service: rules a schema cannot express.

| Concern | Layer |
|---------|-------|
| Type, shape, required fields, format, length, enums | Schema |
| Existence and uniqueness | Service |
| State validity (already cancelled, not active) | Service |
| Cross-field rules (end after start) | Service |
| Authorization context (another user's resource) | Service |
| External checks | Service |

Never repeat a schema rule in the service. Uniqueness is checked in the service for a friendly answer, but a unique index is only added with owner approval and never backs an upsert (`database-conventions`).

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

Errors that leave the server are generic and driven by `errorCode` (`security-baseline`). The `error` text is a fixed generic phrase, the client picks its message from `errorCode`, and no error reveals whether an account or record exists. Internals (SQL text, stack, constraint names, provider payloads) go to the log only.

### Global handler

Register `.onError()` on the root app and `.error({ HttpError })` so the class is known. Handle:

- `VALIDATION` and `PARSE`: 400.
- `NOT_FOUND`: 404.
- `HttpError`: its own status.
- Postgres errors: `23505` and `23503` map to 409, `23502` to 400. The body stays generic: never forward the driver message or constraint name.
- Anything else: 500 with a generic body.

Every error body has the same shape:

```js
{ error, errorCode?, requestId, ts, ...data? }
```

Keep `data` free of internals and of anything the caller sent.

## Auth and guards

One proven pattern. Adapt the mechanism and permission model to the app.

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

### Bitmask RBAC (default permission model)

Use it when an app needs roles beyond "logged in": per-domain bitmasks checked in code, O(1), no DB read per request.

- Split the product into permission domains (content, users, billing, settings, whatever exists). One integer bitmask per domain.
- Cap a domain at 31 flags so the mask stays a signed 32-bit integer and ports across languages (JS bitwise ops, Java `int`, Postgres `integer`). A domain that outgrows 31 flags splits in two.
- Name flags ENTITY_ACTION (`BILLING_VIEW`, `CONTENT_CREATE`), never ACTION_ENTITY, so sorted output groups by resource.
- Tier flags per capability: `X_VIEW` (read) < `X_CREATE` (mutate) < `X_MANAGE` (elevated). A role gets the lowest tier that suffices.
- `ROLE_PERMISSIONS` maps `roleId` to `{ DOMAIN: mask }` and is frozen. The super-admin role has every bit in every domain.
- Refuse any change that would leave no holder of the top role, and say why.

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

Enforcement is dual: backend guards are the security boundary. The frontend imports the same flag and role constants from the shared package only to hide UI and shape messages. Change both sides together.

### Global context

`.derive({ as: "global" }, ...)` injects per-request context such as `requestId`. Use the client IP only for security log lines, never as a rate-limit key.

## Response shapes

| Pattern | Shape |
|---------|-------|
| Single resource | `{ entity: data }` |
| List | `{ entities: [...], total, page, limit }` |
| Creation | `status(201, { entity: data })` |
| Simple action | `{ success: true, message: "..." }` |
| Error | `{ error, errorCode?, requestId, ts }` |

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
