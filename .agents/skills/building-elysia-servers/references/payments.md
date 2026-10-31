# Payment integration

Suggestions, not rules. This is a proven shape to reach for. Adapt it to the provider count, volume and product, and propose deviations openly.

## Provider abstraction

- Define a `PaymentProvider` interface, one adapter per provider and a registry (`registerProvider` / `getProvider`). Adding a provider means implementing the interface and registering it. Services never name a concrete provider.
- Adapters turn provider payloads into the app's normalized events and entities, so nothing downstream knows provider formats.
- Secret keys stay on the server. Only publishable keys reach the frontend, through runtime config.
- Every provider call goes through the timeout and circuit breaker in `ops.md`.

## Webhooks and reconciliation

- Webhooks are the main signal but never the only one. Providers stop retrying after about a day, so a reconcile engine guarantees convergence.
- Deficit-driven reconciliation: SQL selectors find provably stale rows (paid but undated periods, actives past their period end), and only those rows trigger provider calls. Tier 1 is one cheap fetch per stale row. Tier 2 is a windowed replay of provider records.
- One code path: reconciliation applies changes by building normalized events and feeding them to the same handler webhooks use, so the two cannot diverge.
- Verify the signature before processing. Log event ids and outcomes, never payloads with personal or card data.
- Process each event once: claim it with a conditional atomic update on the entity and check provider plus event id (`writing-database-code`). A unique index on the event id needs owner approval and must not back an upsert. With a key-value store, also hold a per-entity semaphore while processing (`redis.md`).

## Provenance and overrides

- Record who, when and how (webhook, sync, manual) as a small JSON object in the entity's `details` jsonb.
- A manual admin override sets a flag in `details` that stops non-forced reconciliation from overwriting it.
