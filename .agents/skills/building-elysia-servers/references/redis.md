# Redis / Valkey

Only when the project has a key-value store (Redis and Valkey share commands and client libraries). Nothing in the core skill depends on it.

## Key prefixes

Three concerns, three prefixes, never mixed:

| Prefix (example) | Concern | Owner |
|------------------|---------|-------|
| `APP:` | Cache and short-lived state: links, counters | cache helper |
| `Q` | Queue library keys only (BullMQ `prefix`) | queue definitions and workers |
| `SEM:` | Semaphores and locks | semaphore helper |

Separate prefixes keep flushes, monitoring and eviction safe: clearing the cache never touches queue or lock state.

## Cache

- Short-lived tokens (emailed links, one-time codes): `SET` with a TTL, single use, deleted on read.
- Emailed link rules: `securing-applications`.
- Counters: `INCR` with expiry for windowed limits, such as a daily email cap.
- The cache is disposable. The app stays correct, only slower, when it is empty or the store is down.

## Distributed semaphore

- Mutex: one holder per key, such as one claim per `entityType:entityId`.
- Permit: up to N holders at once.

Rules:

- Acquire with a TTL so a crashed holder cannot deadlock the key.
- Event processing claims a mutex per `entityType:entityId`. If acquisition fails, re-queue with backoff instead of blocking.
- The semaphore guards entry. The DB conditional update (`writing-database-code`) is what guarantees a row is processed once. Use both when external side effects are involved.

## Rate limiting

Store-backed counters exposed as named tiers (`"strict"`, `"moderate"`) that route middleware consumes (see `http.md`).

- Key contents: `securing-applications`.
- Unauthenticated routes get a lower tier.
- Count before any database read where that is cheap.
- Keep the limits in the application.

## Common mistakes

- Queue keys under the cache prefix, or the reverse.
- A semaphore without a TTL.
- Treating the cache as a source of truth.
- Importing these helpers into a project with no key-value store.
