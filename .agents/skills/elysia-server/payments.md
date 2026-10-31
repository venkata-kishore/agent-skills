# Payment Integration Patterns

Suggestions, not enforcement — a proven shape to reach for when payment work comes up. Adapt to the provider count, volume, and product; propose deviations openly.

## Provider Abstraction

- Define an abstract `PaymentProvider` interface, one adapter per provider, and a registry (`registerProvider` / `getProvider`). Adding a provider = implement the interface + register it; services never reference a concrete provider.
- Normalize at the adapter boundary: adapters translate provider payloads into the app's normalized event and entity shapes, so nothing downstream knows provider formats.
- Keys via env config: secret keys stay server-side; only publishable keys reach the frontend (via runtime config).

## Webhooks + Reconciliation

- Webhooks are the primary signal but never the only one — providers stop retrying after roughly a day, so a reconcile engine is the convergence guarantee.
- **Deficit-driven reconciliation**: SQL selectors find provably stale rows (paid-but-undated periods, actives past their period end), and only those rows trigger provider API calls. Tier 1 = one cheap fetch per stale row; Tier 2 = a windowed replay of provider records.
- **One code path**: reconciliation applies changes by synthesizing normalized events through the same handler webhooks use — sync and webhook cannot diverge.
- Verify webhook signatures before processing. Log event ids and outcomes, never payloads with PII or card data.
- Idempotency: dedup webhook events by provider + event id (see `database-conventions`); claim a per-entity semaphore while processing when a key-value store is available (see `redis.md`).

## Provenance & Overrides

- Record who/when/how (webhook vs sync vs manual) as a small JSON object in the entity's `details` jsonb.
- An admin manual override sets a flag in `details` that blocks non-forced reconciliation from overwriting it.
