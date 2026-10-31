# Redis / Valkey Patterns

Applies only when the project uses a key-value store (Redis or Valkey — same commands, same client libraries). If the project has none, skip this file; nothing in the core skill depends on it.

## Key Namespacing

Keep the three concerns under three separate key prefixes — never mix them:

| Prefix (example) | Concern | Owner |
|------------------|---------|-------|
| App prefix (e.g. `APP:`) | Application cache and keys — magic links, counters, short-lived state | cache helper module |
| Queue prefix (e.g. `Q`) | Queue library keys only (BullMQ `prefix`) | queue definitions + workers |
| Semaphore prefix (e.g. `SEM:`) | Distributed semaphores / locks | semaphore helper module |

Separate prefixes keep flushes, monitoring, and eviction policies safe: clearing the cache must never touch queue or lock state.

## Cache Patterns

- Short-lived tokens (magic links, OTPs): `SET` with TTL, single-use — delete on read.
- Counters: `INCR` with expiry for daily or windowed limits (e.g. a daily email send cap).
- The cache is disposable. The app must stay correct — slower, not wrong — when the cache is empty or the store is down.

## Distributed Semaphore (Concurrency Claims)

Two modes:

- **Mutex**: one holder per key — e.g. one claim per `entityType:entityId`.
- **Permit**: up to N concurrent holders.

Rules:

- Always acquire with a TTL so a crashed holder cannot deadlock the key.
- Webhook/event processing claims a mutex per `entityType:entityId`; if acquisition fails, re-queue the job with backoff instead of blocking.
- The semaphore guards entry; the DB-level atomic claim (conditional update — see `database-conventions`) is what actually guarantees a row is processed once. Use both when external side effects are involved.

## Rate Limiting

Store-backed counters per identity or IP, exposed as named tiers (e.g. `"strict"`, `"moderate"`) that route middleware consumes — see `http.md` guards.

## Common Mistakes

- Using the cache prefix for queue keys (or vice versa) — the three prefixes stay separate.
- Acquiring a semaphore without a TTL.
- Treating the cache as a source of truth instead of a disposable accelerator.
- Assuming a key-value store exists — check the project before importing these helpers.
