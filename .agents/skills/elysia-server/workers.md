# Background Workers & Queue Patterns

## Architecture

```
Services (API server) → Queue.add() → Redis → Worker processes → DB / external APIs
```

Workers run as a **separate Bun process** from the API server. They share the `@pkg/server-common` package (DB client, queries, Redis, logger) but have their own entry point and env config.

## Queue Setup

Queues are defined in a shared package (imported by both server and workers):

```js
import { Queue } from "bullmq";

const defaultOpts = {
  prefix: "UQ",
  connection: { url: process.env.REDIS_URL },
};

export const emailQueue = new Queue("email", defaultOpts);
export const billingQueue = new Queue("billing", defaultOpts);
export const standardQueue = new Queue("standard", defaultOpts);
export const cleanupQueue = new Queue("cleanup", defaultOpts);
export const logisticsQueue = new Queue("logistics", defaultOpts);
```

### Retry Policy (Declared at Queue Creation)

- User-visible or external-effect jobs (email, notifications, payment follow-ups): retries with exponential backoff.
- Idempotent sweeps and scheduled jobs: 1 attempt — the next scheduled run is the retry.

```js
export const emailQueue = new Queue("email", {
  ...defaultOpts,
  defaultJobOptions: { attempts: 3, backoff: { type: "exponential", delay: 30_000 } },
});
```

## Dispatching Jobs from Services

Services add jobs to queues — never directly from routes:

```js
import { emailQueue } from "@pkg/common/queue";

// Template-based email
await emailQueue.add("send-email", {
  template: "welcome",
  to: user.email,
  data: { fullName: user.fullName },
});

// Generic job
await billingQueue.add("process-refund", { subscriptionId, reason });
```

## Worker File Structure

```
workers/src/
  index.js                    # Entry: starts all workers + /healthz endpoint
  config/
    env.js                    # Worker-specific env validation
    redis.js                  # Redis connection opts
  email/
    email.worker.js           # Worker + template dispatch
    email.templates.js        # Template builder functions
    templates/*.html          # Raw HTML templates
  billing/billing.worker.js
  standard/standard.worker.js
  cleanup/cleanup.worker.js
  logistics/logistics.worker.js
  scheduler/scheduler.worker.js
```

## Worker Pattern

```js
import { Worker } from "bullmq";
import { createLogger } from "@pkg/common/logger";
import { defaultRedisOpts } from "../config/redis.js";

const log = createLogger("worker:domain");

const domainWorker = new Worker(
  "queue-name",
  async (job) => {
    // job.name = job type, job.data = payload
    switch (job.name) {
      case "do-thing":
        await handleDoThing(job.data);
        break;
      default:
        log.warn({ jobName: job.name }, "unknown job");
    }
  },
  { ...defaultRedisOpts, concurrency: 5 },
);

domainWorker.on("failed", (job, err) => {
  log.error({ jobId: job?.id, jobName: job?.name, err: err?.message }, "job failed");
});

export { domainWorker };
```

## Scheduler Worker (Cron Dispatcher)

A declarative registry of scheduled jobs. The scheduler worker dispatches to target queues on cron patterns:

```js
const SCHEDULERS = [
  { id: "publish-content", pattern: "*/15 * * * *", queue: "standard", jobName: "publish-content" },
  { id: "clean-tokens", pattern: "0 3 * * *", queue: "cleanup", jobName: "clean-tokens" },
  { id: "sweep-past-due", pattern: "0 */12 * * *", queue: "billing", jobName: "sweep-past-due" },
];
```

Uses BullMQ's `upsertJobScheduler` for idempotent registration. Orphan cleanup removes schedulers not in the current registry.

## Entry Point Pattern

```js
import { workerA } from "./domain-a/a.worker.js";
import { workerB } from "./domain-b/b.worker.js";

// Graceful shutdown
async function shutdown() {
  await Promise.all([workerA.close(), workerB.close()]);
  process.exit(0);
}
process.on("SIGTERM", shutdown);
process.on("SIGINT", shutdown);

// Health check (separate port from API server)
Bun.serve({
  port: Number(process.env.WORKER_HEALTH_PORT) || 3001,
  fetch(req) {
    if (new URL(req.url).pathname === "/healthz") {
      return Response.json({ status: "healthy", timestamp: DateTime.now().toISO() });
    }
    return new Response("Not Found", { status: 404 });
  },
});
```

## Email Worker Specifics

- Template-based: `{ template: "name", to, data }` — builder function produces `{ subject, htmlBody }`
- Daily send limit via Redis counter (auto-delays overflow to next midnight)
- Rate limiter: `{ max: 12, duration: 1000 }` (12 emails/sec)
- Concurrency: 5

## Conventions

| Rule | Details |
|------|---------|
| Job naming | kebab-case: `"send-email"`, `"sweep-past-due"` |
| Logging | `createLogger("worker:domain")` for structured pino logs |
| Error handling | Let BullMQ retry on throw; log in `"failed"` event handler |
| Retry policy | Declared at queue creation — retries + backoff for user-visible jobs, 1 attempt for idempotent sweeps |
| Concurrency | Set per-worker based on I/O profile (email=5, cleanup=1, billing=3) |
| Idempotency | Workers should be safe to retry — use idempotent DB operations |
| No HTTP imports | Workers never import from the server package — shared code lives in common |
| Delayed jobs | `job.moveToDelayed(timestamp, token)` + `throw new DelayedError()` |
