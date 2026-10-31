# Background workers and queues

## Architecture

```text
Services (API server) -> Queue.add() -> Redis -> Worker processes -> DB / external APIs
```

Workers run as a separate Bun process. They share the common package (DB client, queries, Redis, logger) but have their own entry point and env config.

## Queues

Queues are defined in the shared package, imported by server and workers:

```js
import { Queue } from "bullmq";

const defaultOpts = {
  prefix: "UQ",
  connection: { url: process.env.REDIS_URL },
};

export const emailQueue = new Queue("email", defaultOpts);
export const billingQueue = new Queue("billing", defaultOpts);
export const cleanupQueue = new Queue("cleanup", defaultOpts);
```

### Retry policy

Declare it at queue creation.

- Jobs with a user-visible or external effect (email, notifications, payment follow-ups): retries with exponential backoff.
- Idempotent sweeps and scheduled jobs: one attempt. The next scheduled run is the retry.

```js
export const emailQueue = new Queue("email", {
  ...defaultOpts,
  defaultJobOptions: { attempts: 3, backoff: { type: "exponential", delay: 30_000 } },
});
```

## Dispatching

Services add jobs, never routes.

```js
await emailQueue.add("send-email", {
  template: "welcome",
  to: user.email,
  data: { fullName: user.fullName },
});
await billingQueue.add("process-refund", { subscriptionId, reason });
```

## Worker file structure

```text
workers/src/
  index.js                  # starts all workers and /healthz
  config/env.js             # worker env validation
  config/redis.js           # Redis connection opts
  email/email.worker.js     # worker and template dispatch
  email/email.templates.js  # template builders
  billing/billing.worker.js
  cleanup/cleanup.worker.js
  scheduler/scheduler.worker.js
```

## Worker pattern

```js
const log = createLogger("worker:domain");

const domainWorker = new Worker(
  "queue-name",
  async (job) => {
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
```

## Scheduler worker (cron dispatcher)

A declarative registry; the scheduler worker enqueues to target queues on cron patterns.

```js
const SCHEDULERS = [
  { id: "clean-tokens", pattern: "0 3 * * *", queue: "cleanup", jobName: "clean-tokens" },
  { id: "sweep-past-due", pattern: "0 */12 * * *", queue: "billing", jobName: "sweep-past-due" },
];
```

Register with `queue.upsertJobScheduler(id, { pattern }, { name, data, opts })`, which is idempotent. Remove schedulers that are no longer in the registry.

## Entry point

```js
async function shutdown() {
  await Promise.all([workerA.close(), workerB.close()]);
  process.exit(0);
}
process.on("SIGTERM", shutdown);
process.on("SIGINT", shutdown);

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

The health port differs from the API port.

## Email worker

- Template-based payload `{ template, to, data }`; a builder returns `{ subject, htmlBody }`.
- Daily send cap through a Redis counter; overflow is delayed to the next midnight.
- Worker rate limiter `{ max: 12, duration: 1000 }`, concurrency 5.
- The send call to the mail provider uses the timeout and circuit breaker from `ops.md`.
- Emailed links follow `redis.md`: GET never changes state.

## Conventions

| Rule | Detail |
|------|--------|
| Job names | kebab-case: `"send-email"`, `"sweep-past-due"` |
| Logging | `createLogger("worker:domain")`; log failures in the `"failed"` handler |
| Errors | Let BullMQ retry on throw |
| Concurrency | Per worker, by I/O profile (email 5, cleanup 1, billing 3) |
| Idempotency | Every job is safe to retry: idempotent DB operations (`writing-database-code`) |
| Imports | Workers never import from the server package |
| Delayed jobs | `job.moveToDelayed(timestamp, token)` then `throw new DelayedError()` |
