# Logging, config and operational safety

## Logging

```js
import { createLogger } from "@pkg/common/infrastructure/logger";

const log = createLogger("service:subscriptions");
```

`createLogger(name)` returns a pino child logger with `{ module: name }` bound. Name it `"layer:domain"`: `"service:auth"`, `"worker:email"`, `"middleware:rate-limit"`, `"webhooks"`, `"payments"`.

First argument is the bindings object, second is the message:

```js
log.info({ userId, planId }, "checkout initiated");
log.warn({ code: err?.code, err: err?.message }, "payment verification retry failed");
log.error({ jobId: job?.id, code: err?.code, err: err?.message }, "job failed");
```

Log an error's message and code only, never the whole `err` object: it can carry provider payloads, request bodies or personal data.

| Log | Do not log |
|-----|-----------|
| Service events: created, updated, failed | Every DB query |
| Worker jobs: start, success, failure | Request and response bodies |
| Auth events (fields below) | Passwords, tokens, cookie values, private keys, raw personal data |
| External calls: payment, email dispatch | Expected validation failures |
| Unexpected errors with context | Health-check noise |

Security log fields, audit and the rest of the security rules: `securing-applications`.

## Environment config

```js
function required(key, fallback) {
  const val = process.env[key] ?? fallback;
  if (val === undefined) throw new Error(`Missing required environment variable: ${key}`);
  return val;
}

export const env = Object.freeze({
  PORT: Number(process.env.PORT) || 3000,
  IS_PRODUCTION: process.env.NODE_ENV === "production",

  DATABASE_URL: required("DATABASE_URL"),
  REDIS_URL: required("REDIS_URL"),

  JWT_PRIVATE_KEY_PEM: required("JWT_PRIVATE_KEY_PEM"),
  JWT_PUBLIC_KEY_PEM: required("JWT_PUBLIC_KEY_PEM"),

  PAYMENT_KEY_ID: required("PAYMENT_KEY_ID"),
  FRONTEND_URL: process.env.FRONTEND_URL || "http://localhost:3001",
});
```

- `required(key)` with no fallback crashes at startup. The message names only the variable, never its value.
- `required(key, fallback)` uses the fallback only when the variable is absent.
- Freeze the export. Group variables by concern: server, database, redis, auth, external services.
- Coerce numbers inline: `Number(process.env.PORT) || default`.
- The fallback ports here and in the worker health check suit scaffolding only. A real project sets `PORT`, `FRONTEND_URL` and `WORKER_HEALTH_PORT` to its own 88xx block (`~/.agents/instructions/environment.md`).
- Workers have their own `config/env.js` with worker-specific variables.

## Outbound calls

Wrap every call to a dependency (payment provider, email service, any third-party API) with a short timeout and a circuit breaker (rules in `securing-applications`). In Bun, pass `signal: AbortSignal.timeout(ms)` to `fetch`. Workers use the same wrapper: a skipped call throws, and the queue retry policy does the waiting. Keep one timeout per call; if a test needs a timer race, see `writing-tests`.

## Operational safety

- Transactions are short and DB-only (`writing-database-code`).
- Log no request or response bodies unless they are known to be safe.
- Jobs must be idempotent: BullMQ retries failed jobs (`writing-database-code`).
