---
name: database-conventions
version: 2
description: Use when writing or reviewing database code — Drizzle ORM queries, transactions, raw SQL, migrations, schema design, foreign keys, idempotent writes, status conventions, shared constants/enums, ID generation, or DB-to-API mappers
agents: [main_agent, general_purpose]
---

# Database Conventions

Data-layer conventions for the backend: query organization, status semantics, shared constants, ID strategy, and mapping DB rows to API responses.

## File Organization

One file per entity in a shared queries directory: `<entity>.queries.js`. Each exports named async functions.

Feature modules keep route/service/schema files under `modules/<feature>/`; database access belongs in shared query files only when the feature needs persistence.

## Schema Design

Design columns for cheap querying and indexing, not just storage.

- **Enums and statuses are integers, never text.** Map every fixed set of states to numbers and store the smallest type that fits the cardinality — Postgres `smallint` (MySQL `tinyint`) holds far more states than any enum needs. Text values waste space, bloat indexes, and make range/ordering queries awkward.
- **Number states in a meaningful order** so comparisons and sorts are queryable. Follow the negative/zero/positive scheme (see Status Conventions): terminal states negative and ordered by severity (`marked_for_delete: -1`, `deleted: -2`), neutral/initial at `0` (`draft`), active states positive and ordered by liveness (`inactive: 6`, `pending_approval: 7`, `active: 10`). This keeps `status > 0`, `status < 0`, and `ORDER BY status` direct and index-friendly.
- **Don't over-normalize small, static data.** Reference sets that are tiny (a handful of rows) and effectively fixed — roles, tiers, statuses, types — can live in code as shared constants/enums instead of a lookup table (see Constants & Shared Enums). Promote to a table only when the set grows, changes at runtime, or the user asks for it.
- **Default ID and count columns to `bigint`** (always `{ mode: "string" }` in JS — bigint exceeds JS number precision, though it maps one-to-one to `long` in most other languages). Right-size down to `int`/`smallint` only when a table is provably bounded — tiny lookups, fixed reference sets. See ID Generation.
- **No foreign key constraints by default.** House convention for high-performance Postgres: the app layer owns referential integrity — services validate references and delete/update related rows inside the same transaction boundary, so FK checks would only add write-time overhead and lock contention. Add FK constraints only when the user explicitly asks for them.
- **No unique indexes for idempotency.** "Only once" guarantees are app-level: check-then-insert inside a transaction (see `drizzle.md`). Exception: append-only event-dedup tables (e.g. webhook events keyed by provider + event id) may carry a unique target for `ON CONFLICT DO NOTHING` inserts.
- **Keep queryable data in real columns, not JSON.** Treat `jsonb` columns as opaque blobs read and written as a whole (settings, metadata, snapshots, external payloads). Any field you filter, join, sort, or index on belongs in its own column — querying inside JSON means fetching the blob and digging into it per row. Postgres `jsonb` *can* be indexed (GIN, or an expression index on `data->>'field'`) when you genuinely must query a path, but reach for a promoted column first.

## Supporting References

- Use `drizzle.md` for Drizzle ORM query mechanics — the standard query builder, transactions, raw SQL, dynamic `WHERE`, JSONB merge, partial-update guards, and key rules.

## Migrations

Determine the migration mode before touching anything — the two modes are handled very differently. Check for a migrations directory, `drizzle.config.*`, and package scripts first.

| Mode | Source of truth | Workflow |
|------|-----------------|----------|
| Tool-managed | Drizzle schema file | Edit schema → `drizzle-kit generate` → tooling applies; never hand-edit generated SQL |
| Manual SQL | Hand-numbered SQL files | Write `00N_name.sql` and update the schema file in the same change; a human applies it |

Both modes:

- Write idempotent DDL wherever possible (`IF NOT EXISTS` / `IF EXISTS`, guarded seed inserts) so a re-run neither fails nor duplicates.
- Seed data lives in its own numbered seed migration, never ad-hoc inserts.
- The schema file and the applied SQL never drift — same change, same review.

Manual-mode discipline:

- Apply with a fail-fast flag: `psql "$DATABASE_URL" -v ON_ERROR_STOP=1 -f migrations/00N_name.sql` — plain `psql -f` exits 0 even when statements fail.
- DDL runs as the table-owner role; the app runtime role carries DML grants only.
- Confirm with the user before destructive schema changes.

## Status Conventions

All status integers follow a negative/zero/positive semantic:

| Range | Meaning | Examples |
|-------|---------|---------|
| Negative (< 0) | Terminal / dead / blocked | `DELETED: -10`, `CANCELLED: -5`, `SUSPENDED: -3` |
| Zero | Neutral / created / pending | `CREATED: 0`, `PENDING: 0`, `DRAFT: 0` |
| Positive (> 0) | Alive / active / progressing | `ACTIVE: 10`, `PAST_DUE: 5`, `FUTURE_CANCEL: 7` |

Use range checks where appropriate:

- `status > 0` → entity is alive / has access.
- `status < 0` → entity is terminated.
- `status === 0` → entity is in initial/pending state.

Keep the magnitudes ordered as you add new statuses — the ordering is what makes range filters and `ORDER BY status` direct and index-friendly.

## Constants & Shared Enums

Constants live in a shared package importable by both frontend and backend:

```text
common/src/constants/
  statuses.js
  roles.js
  tiers.js
  billing-cycles.js
  tokens.js
```

Simple numeric enums:

```js
export const SUBSCRIPTION_STATUS = Object.freeze({
  EXPIRED: -10,
  CANCELLED: -5,
  CREATED: 0,
  PAUSED: 4,
  PAST_DUE: 5,
  ACTIVE: 10,
});
```

Rich descriptors carry `id` and `label`:

```js
class UserRole {
  constructor(id, label) {
    this.id = id;
    this.label = label;
    Object.freeze(this);
  }

  toString() {
    return this.label;
  }
}

export const ROLES = Object.freeze({
  SUPER_ADMIN: new UserRole(1, "Super Admin"),
  MEMBER: new UserRole(20, "Member"),
});
```

Usage: `ROLES.MEMBER.id` for DB storage, `ROLES.MEMBER.label` for display.

Import convention:

```js
import { SUBSCRIPTION_STATUS, ROLES } from "@pkg/common";
```

Never hardcode magic numbers — always reference the constant.

## ID Generation

Choose an ID strategy based on app scale and requirements.

One approach: snowflake-style IDs for high-traffic tables such as users, subscriptions, and orders:

- Upper bits = milliseconds since a custom epoch.
- Lower bits = atomic counter using Redis `INCR` with local fallback.
- Result is time-sortable, unique across instances, and stored as `bigint` with `{ mode: "string" }`.

For small reference tables such as plans, subjects, and roles, use Postgres auto-increment identity columns.

Default ID columns to `bigint` (and always for snowflake IDs). Drop to `int`/`smallint` only for provably bounded tables — tiny lookups, fixed reference sets — never as a squeeze on a table that could grow.

Pre-generate IDs before insert when the ID must be referenced within the same transaction across multiple tables.

## Mappers (DB → API Response)

Transform DB row shapes into API-safe response objects. Locate mappers in `lib/mappers.js`.

```js
export function mapUserToProfile(user, security = null) {
  if (!user) return null;
  return {
    id: user.id,
    email: user.email,
    fullName: user.fullName,
    roleId: user.roleId,
    status: user.status,
    createdAt: user.createdAt,
    hasPassword: security ? !!security.passwordHash : undefined,
    hasGoogle: security ? !!security.googleSsoId : undefined,
  };
}
```

Conventions:

- Name mappers `mapXToY`.
- Return `null` if input is null.
- Use boolean presence-check fields from companion rows; set `undefined` to omit from JSON when unavailable.
- Use mappers in route handlers: `return { user: mapUserToProfile(row, secRow) }`.
- Keep DB internals, password hashes, and internal flags out of API responses.

## Quick Reference

| Need | Pattern |
|------|---------|
| New entity queries | Add `<entity>.queries.js` with named async functions in the shared DB package |
| DB single row | `const [row] = await db.select()...limit(1)`; return `row ?? null` |
| Insert/update/delete | Always end with `.returning()` |
| DB transaction | Pass `tx` parameter through query calls; `await db.transaction(async (tx) => ...)` |
| Status value | Reference a shared constant; negative = terminal, 0 = pending, positive = active |
| Enum/status column | Numeric `smallint`, ordered (negative terminal → positive active), never text |
| ID column type | `bigint` default; `int`/`smallint` only for provably bounded lookup tables |
| Query a field | Real column, not a field buried inside a `jsonb` blob |
| Small static set | Keep in code as constants; add a lookup table only when it grows or changes |
| BigInt IDs | Use `{ mode: "string" }` and handle as strings |
| DB → API shape | Map rows with `mapXToY` before returning |
| Migration | Determine mode first: tool-managed (drizzle-kit) vs manual numbered SQL |
| Idempotent write | Check-then-insert inside a transaction — not a unique index |
| Claim a row | Conditional update `WHERE status = <expected>` + `.returning()`; a returned row = claim won |
| FK constraints | Skip by default — app layer owns integrity; add only when explicitly requested |

## Common Mistakes

- Forgetting `.returning()` on insert/update/delete queries.
- Not passing `tx` through transaction call chains.
- Calling external APIs inside DB transactions.
- Hardcoding magic numbers instead of referencing shared constants.
- Returning raw DB rows (with internal flags or password hashes) instead of mapping them.
- Storing enums or statuses as text instead of ordered integers.
- Putting queryable or filterable fields inside a `jsonb` blob instead of their own columns.
- Creating a lookup table for a handful of fixed records that belong in code.
- Dropping to raw SQL when the Drizzle builder would do the job.
- Adding FK constraints or idempotency unique indexes unprompted — integrity and dedup are app-layer by convention.
- Applying manual SQL migrations without `-v ON_ERROR_STOP=1`, or running DDL as the app runtime role instead of the table owner.
- Assuming the migration mode instead of checking whether the project is tool-managed or manual SQL.
