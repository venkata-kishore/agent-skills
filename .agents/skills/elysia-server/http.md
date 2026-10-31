# HTTP, Validation, Auth, and Error Patterns

## Input Validation

Validation happens at two layers with distinct responsibilities.

### Layer 1: Schema Validation (Route Level)

Elysia's TypeBox schemas handle structural/type validation: shape, types, and basic constraints. Apply schemas declaratively on routes via `{ body: Schema }`, `{ query: Schema }`, or `{ params: Schema }`.

```js
export const CreateThingBody = t.Object({
  name: t.String({ minLength: 1, maxLength: 200 }),
  email: t.String({ format: "email" }),
  priority: t.Integer({ minimum: 1, maximum: 5 }),
  tags: t.Optional(t.Array(t.String(), { minItems: 1 })),
  status: t.Union([t.Literal("draft"), t.Literal("published")]),
  metadata: t.Optional(t.Record(t.String(), t.String())),
});

.post("/api/things", handler, { body: CreateThingBody })
.get("/api/things", handler, { query: ListThingsQuery })
```

Schema failures are caught by Elysia and routed to error middleware as `VALIDATION` → 400.

Common TypeBox patterns:

- `t.String({ format: "email" })` — email format.
- `t.String({ minLength: N, maxLength: M })` — bounded strings.
- `t.Optional(t.Union([t.String(), t.Null()]))` — nullable optional.
- `t.Union([t.Literal("A"), t.Literal("B")])` — enum via literals.
- `t.Integer({ minimum: N, maximum: M })` — bounded integers.
- `t.Array(t.String(), { minItems: 1 })` — non-empty arrays.
- `t.Record(t.String(), t.Object({...}))` — dynamic key objects.

### Layer 2: Business Validation (Service Level)

Services handle semantic/business validation that schemas cannot express: uniqueness, existence checks, state transitions, cross-field rules, external lookups, and authorization context.

```js
export async function createUser(data) {
  const existing = await findUserByEmail(data.email);
  if (existing) throw Conflict("Email already in use", "EMAIL_TAKEN");

  if (isDisposableEmail(data.email)) throw BadRequest("Disposable emails not allowed", "DISPOSABLE_EMAIL");

  if (data.endDate && data.endDate <= data.startDate) {
    throw BadRequest("End date must be after start date", "INVALID_DATE_RANGE");
  }

  return await insertUser(data);
}
```

### When to Validate Where

| Concern | Layer | Example |
|---------|-------|---------|
| Type/shape correctness | Schema | string, integer, required fields |
| Format constraints | Schema | email format, min/max length, enums |
| Existence/uniqueness | Service | "email already taken", "user not found" |
| State validity | Service | "subscription not active", "already cancelled" |
| Cross-field rules | Service | "end date must be after start date" |
| Authorization context | Service | "cannot edit another user's resource" |
| External system checks | Service | disposable email lists, rate limits |

Never duplicate schema validation in service logic. If schema enforces `minLength: 1`, do not re-check for empty string in the service.

## Error Handling

### Error Factories

```js
export class HttpError extends Error {
  constructor(status, message, errorCode, data) { ... }
}

export function BadRequest(message, errorCode, data) { return new HttpError(400, ...); }
export function Unauthorized(message, errorCode, data) { return new HttpError(401, ...); }
export function Forbidden(message, errorCode, data) { return new HttpError(403, ...); }
export function NotFound(message, errorCode, data) { return new HttpError(404, ...); }
export function Conflict(message, errorCode, data) { return new HttpError(409, ...); }
export function TooManyRequests(message, errorCode, data) { return new HttpError(429, ...); }
```

Usage in services:

```js
throw BadRequest("Email already in use", "EMAIL_TAKEN");
```

Do not use `new` with error factories.

### Global Error Middleware

Register on the root app via `.onError()`. Handle:

- `VALIDATION` → 400.
- `NOT_FOUND` → 404.
- `PARSE` → 400.
- `HttpError` → uses the error's status.
- Postgres errors → auto-mapped, such as `23505` → 409, `23503` → 409, `23502` → 400.
- Unknown errors → 500.

All error responses share this shape:

```js
{ error, errorCode?, requestId, ts, ...data? }
```

## Auth & Guards

This section describes one proven pattern. Adapt the auth mechanism and permission model to the app.

### Global Auth Resolve + Per-Route Guards

Use Elysia's `.resolve({ as: "global" })` to extract auth state once and inject it into all handlers. Individual routes decide if auth is required.

```js
export const authMiddleware = new Elysia({ name: "auth.middleware" })
  .resolve({ as: "global" }, async ({ headers, cookie }) => {
    const user = await verifyToken(cookie.token?.value);
    return { user };
  });
```

### Guard Factories as `beforeHandle`

Create composable guard functions that return Elysia `beforeHandle` hooks:

```js
export function requireAuth() {
  return ({ user }) => {
    if (!user) throw Unauthorized("Authentication required", "AUTH_REQUIRED");
  };
}

export function requirePermission(domain, flag) {
  return ({ user }) => {
    if (!user) throw Unauthorized("Authentication required", "AUTH_REQUIRED");
    if (!canDo(user.roleId, domain, flag)) throw Forbidden("Insufficient permissions", "FORBIDDEN");
  };
}
```

Apply per-route or grouped:

```js
.get("/api/admin/users", handler, { beforeHandle: [requirePermission("USR", USR.USERS_VIEW)] })

.guard({ beforeHandle: [requireAuth()] }, (app) =>
  app.get("/api/me", handler).patch("/api/me", handler),
)
```

### Bitmask RBAC (Go-To Permission Model)

The default permission model when an app needs roles beyond "logged in": per-domain bitmasks checked in code — O(1), no DB lookup per request.

- Split the product into permission domains (content, users, billing, settings — whatever the product actually has). Each domain is one integer bitmask.
- **Cap a domain at 31 flags** so the mask stays inside a signed 32-bit integer and ports cleanly across languages (JS bitwise ops, Java `int`, Postgres `integer`). A domain that outgrows 31 flags splits into two domains.
- Flag names are **ENTITY_ACTION** (`BILLING_VIEW`, `CONTENT_CREATE`) — never ACTION_ENTITY; sorted output then groups flags by resource.
- Tier flags per capability: `X_VIEW` (read) < `X_CREATE` (mutate) < `X_MANAGE` (elevated). A role gets the lowest tier that suffices.
- `ROLE_PERMISSIONS` maps `roleId → { DOMAIN: mask, ... }` (frozen); the super-admin role gets all bits in every domain.

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

Enforcement is dual-layer: backend guards (`requirePermission(domain, flag)`) are the security boundary; the frontend imports the same flag and role constants from the shared package only to hide UI and shape messages. When access rules change, update both sides together.

### Global Context via `derive` / `resolve`

Use `.derive({ as: "global" })` or `.resolve({ as: "global" })` to inject cross-cutting context available in handlers:

```js
.derive({ as: "global" }, () => ({ requestId: generateId() }))

.derive({ as: "global" }, ({ headers, server }) => ({ clientIp: extractIp(headers) }))
```

## Response Conventions

| Pattern | Shape |
|---------|-------|
| Single resource | `{ entity: data }` |
| List | `{ entities: [...], total, page, limit }` |
| Creation (201) | `status(201, { entity: data })` |
| Simple action | `{ success: true, message: "..." }` |
| Error | `{ error, errorCode?, requestId, ts }` |

## App Assembly

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

Middleware order matters:

1. Logging/request-id.
2. Error handler.
3. Auth.
4. Rate limiting.
5. Route modules.
