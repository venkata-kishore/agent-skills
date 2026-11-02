# Background workers and queues

Rules (side effects via queue from the service, idempotent jobs, workers never import the server package): `building-backends`. This file holds the BullMQ mechanics.

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
  prefix: "Q",
  connection: { url: process.env.REDIS_URL },
};

export const queueA = new Queue("queue-a", defaultOpts);
export const queueB = new Queue("queue-b", defaultOpts);
```

### Retry policy

Declare it at queue creation (which jobs retry: `building-backends`).

```js
export const queueA = new Queue("queue-a", {
  ...defaultOpts,
  defaultJobOptions: { attempts: 3, backoff: { type: "exponential", delay: 30_000 } },
});
```

## Dispatching

Services add jobs.

```js
await queueA.add("do-thing", { thingId, reason });
await queueB.add("do-other-thing", { userId });
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
  { ...defaultOpts, concurrency: 5 },
);

domainWorker.on("failed", (job, err) => {
  log.error({ jobId: job?.id, jobName: job?.name, err: err?.message }, "job failed");
});
```

## Scheduler worker (cron dispatcher)

A declarative registry; the scheduler worker enqueues to target queues on cron patterns.

```js
const SCHEDULERS = [
  { id: "do-thing-daily", pattern: "0 3 * * *", queue: "queue-a", jobName: "do-thing" },
  { id: "sweep-stale", pattern: "0 */12 * * *", queue: "queue-b", jobName: "sweep-stale" },
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
  port: Number(process.env.WORKER_HEALTH_PORT) || 4001,
  fetch(req) {
    if (new URL(req.url).pathname === "/healthz") {
      return Response.json({ status: "healthy", timestamp: DateTime.now().toISO() });
    }
    return new Response("Not Found", { status: 404 });
  },
});
```

The port above is an example. It must differ from the API port and from the frontend dev server's.

## Email worker

- Template-based payload `{ template, to, data }`; a builder returns `{ subject, htmlBody }`.
- Daily send cap through a Redis counter; overflow is delayed to the next midnight.
- Worker rate limiter `{ max: 12, duration: 1000 }`, concurrency 5.
- The send call to the mail provider uses the timeout and circuit breaker from `ops.md`.
- Emailed link rules: `securing-applications`.

## Conventions

| Rule | Detail |
|------|--------|
| Job names | kebab-case: `"send-email"`, `"sweep-stale"` |
| Logging | `createLogger("worker:domain")`; log failures in the `"failed"` handler |
| Errors | Let BullMQ retry on throw |
| Concurrency | Per worker, by I/O profile (for example 5 for I/O-heavy, 1 for sweeps) |
| Idempotency | Every job is safe to retry (`building-backends`) |
| Delayed jobs | `job.moveToDelayed(timestamp, token)` then `throw new DelayedError()` |
