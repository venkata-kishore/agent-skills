# Logging, Environment, and Operational Patterns

## Logging

### Setup

```js
import { createLogger } from "@pkg/common/infrastructure/logger";

const log = createLogger("service:subscriptions");
```

`createLogger(name)` returns a pino child logger with `{ module: name }` bound so every log line carries the module field automatically.

### Naming Convention

Format: `"layer:domain"`.

Examples:

- `"service:auth"`
- `"service:billing"`
- `"worker:email"`
- `"worker:scheduler"`
- `"middleware:rate-limit"`
- `"webhooks"`
- `"payments"`

### Usage

```js
log.info({ userId, planId }, "checkout initiated");
log.warn({ err }, "payment verification retry failed");
log.error({ jobId: job?.id, err: err?.message }, "job failed");
log.debug({ token }, "refresh token rotated");
```

First arg = structured bindings object, second arg = message string.

### What to Log

| Log | Don't Log |
|-----|-----------|
| Service-level events: created, updated, failed | Every DB query |
| Worker job processing: start, success, failure | Request/response bodies |
| Auth events: login, logout, token rotation | Passwords, tokens, PII |
| External API calls: payment, email dispatch | Expected validation failures |
| Unexpected errors with context | Noise from health checks |

## Environment & Config

### Pattern

```js
function required(key, fallback) {
  const val = process.env[key] ?? fallback;
  if (val === undefined) throw new Error(`Missing required environment variable: ${key}`);
  return val;
}

export const env = Object.freeze({
  PORT: Number(process.env.PORT) || 43000,
  NODE_ENV: process.env.NODE_ENV || "development",
  IS_PRODUCTION: process.env.NODE_ENV === "production",

  DATABASE_URL: required("DATABASE_URL"),
  REDIS_URL: required("REDIS_URL"),

  JWT_PRIVATE_KEY_PEM: required("JWT_PRIVATE_KEY_PEM"),
  JWT_PUBLIC_KEY_PEM: required("JWT_PUBLIC_KEY_PEM"),

  RAZORPAY_KEY_ID: required("RAZORPAY_KEY_ID"),
  FRONTEND_URL: process.env.FRONTEND_URL || "http://localhost:3001",
});
```

### Rules

- `required(key)` with no fallback crashes at startup if missing.
- `required(key, fallback)` uses fallback only if the env var is absent.
- Freeze the entire export with `Object.freeze()`.
- Group variables by concern: server, database, redis, auth, external services.
- Use inline numeric coercion: `Number(process.env.PORT) || default`.
- Workers have their own `config/env.js` with worker-specific vars.

## Runtime & Tooling Reminders

- Bun is the runtime and package manager.
- Use `bun:test` for backend tests.
- Default tests are unit-level and infra-free: test functions/methods (mappers, validators, permission checks, status transitions, service logic) without a live DB, Redis/Valkey, queue, or network. Keep logic in plain functions with I/O at the edges so this stays possible.
- Integration tests against real infra only when the user explicitly asks.
- Use Bun native APIs where applicable.
- Use `oxlint` and `oxfmt`; print width is 100.
- Prefer JavaScript for rapid prototyping unless the project explicitly chooses TypeScript.
- Respect existing formatting in unchanged files.

## Operational Safety

- Keep DB transactions short and DB-only.
- Do not call external APIs inside DB transactions.
- Avoid logging request/response bodies unless explicitly safe.
- Never log passwords, tokens, private keys, or raw PII.
- Workers should be idempotent because BullMQ retries failed jobs.
