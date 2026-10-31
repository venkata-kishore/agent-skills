---
name: writing-database-code
description: Use when writing or reviewing database code, schema design, migrations, indexes, unique constraints, status or enum columns, ID generation, idempotent writes, transactions, JSON columns, raw SQL, shared constants, or DB-to-API mappers. Any SQL database (PostgreSQL assumed where it matters); Drizzle ORM and PostgreSQL specifics are in references.
---

# Database Conventions

Data-layer rules: schema, indexes, statuses, constants, IDs, migrations, mappers. Any SQL database; PostgreSQL is assumed where it matters.

## References

- `references/drizzle.md`: open when the project uses Drizzle ORM (query builder, relations v2, client wiring, transactions, raw SQL, key rules).
- `references/postgres.md`: open when creating a PostgreSQL database or cluster, choosing collation, or reasoning about transaction isolation.

## Organization

One file per entity in a shared queries directory: `<entity>.queries.js`, each exporting named async functions. Feature modules keep route, service and schema files under `modules/<feature>/`; database access goes in the shared query files.

## Schema

- **Known-value fields are small-integer codes, never text.** Use the smallest type that fits (`smallint`). Number codes without gaps. Emit them to the frontend as generated constants.
- **Bigint ids everywhere**, serialised as strings for JavaScript (bigint exceeds JS number precision). Counts and other numeric columns may be right-sized.
- **Keep the schema simple.** Fewer tables; a column instead of a side table where queries allow. Split only when a query needs it. Separate preferences (visible to everyone) from settings (access-controlled).
- **Small static sets live in code** as shared constants, not a lookup table, until the set grows or changes at runtime.
- **No foreign keys by default.** The app layer owns referential integrity and writes related rows in one transaction. Add FKs only when the owner asks.
- **No CHECK constraints, no DEFAULT clauses except `created_at`, no triggers.** Rules live in code with tests. An exception needs the owner's explicit approval.
- **Real columns for anything queried.** Filter, join, sort or index fields are columns, not JSON keys.
- **JSON columns** (`jsonb`) are nullable with no default and read and written as a whole. Stored keys are short, declared as constants, with a comment explaining the abbreviations. API bodies keep full names.
- **Canonicalise at write** (lower-case emails and hostnames) so lookups use plain equality and plain indexes, not case-insensitive collations or `lower()` in queries.
- **Text limits** count characters: 2,000 by default, smaller for short fields, 5,000 for bodies, unless decided otherwise.
- **Time is UTC.** Servers and database run in UTC; set the process default zone at boot before anything else initialises.
- **Sort order** for typed names is deterministic: code-point order unless the owner asks for linguistic order. Record the choice.
- **Archive instead of deleting reference data.** Archiving blocks new use; in-flight work runs to completion.

## Indexes and Uniqueness

- No index until a named query needs it. Name that query when you add the index.
- Prefer one composite index over a single-column index with the same leading column.
- Every new unique index needs the owner's approval, including event-dedup tables.
- A unique index is an integrity constraint, never the basis of an upsert. Do not use `ON CONFLICT` to get idempotency.

## Idempotent Writes

Every job and write path is safe to repeat: separate a side effect from recording it, skip settled rows, re-check the condition inside the guarded `UPDATE`, treat an already-missing object as done.

Check-then-insert is not safe alone: two concurrent transactions can both see nothing and both insert. Serialise first, either by locking the owning row (a guarded update or `SELECT ... FOR UPDATE`) before the check, or by claiming with a conditional `UPDATE ... WHERE status = <expected>` and treating a returned row as a won claim. Example in `references/drizzle.md`.

## Tenancy

Anything that runs outside a web request (jobs, workflows, queues, object keys, exports, log lines) carries the tenant id with every entity id. Never encode a location (shard, region, database) in an id. Worker ids are unique across every instance in every region.

Tenancy without row-level security: one scoped data-access gateway, and every mutation writes exactly one audit row in the same transaction.

## Status Codes

One convention across every table:

| Code | Meaning |
|------|---------|
| `0` | base marker (row exists, nothing else known) |
| `1` | draft |
| `2` | ended normally |
| `3` | archived |
| `> 3` | live states, ordered by liveness |
| `< 0` | removable without real loss |

Documented exceptions only. Range checks work directly: `status > 3` is live, `status < 0` is removable, `ORDER BY status` follows the lifecycle.

## Constants

Constants live in a shared package imported by backend and frontend. Never hardcode magic numbers.

```js
export const JOB_STATUS = Object.freeze({
  DISCARDED: -1,
  BASE: 0,
  DRAFT: 1,
  DONE: 2,
  ARCHIVED: 3,
  QUEUED: 4,
  PROCESSING: 5,
});
```

Rich descriptors (roles, tiers) carry `id` and `label` in a frozen object: `ROLES.MEMBER.id` for storage, `.label` for display.

## ID Generation

Choose by scale. Snowflake-style ids suit high-traffic tables: upper bits are milliseconds since a custom epoch, lower bits a counter (Redis `INCR` with local fallback), stored as `bigint`. Small reference tables may use identity columns. Pre-generate ids when several tables in one transaction must reference them.

## Migrations

Determine the mode first (migrations directory, ORM config, package scripts).

| Mode | Source of truth | Workflow |
|------|-----------------|----------|
| Tool-managed | ORM schema file | Edit schema, run the generator, tooling applies; never hand-edit generated SQL |
| Manual SQL | Numbered SQL files | Write `00N_name.sql` and update the schema file in the same change; a human applies it |

- Write idempotent DDL (`IF NOT EXISTS` / `IF EXISTS`, guarded seed inserts). Seed data lives in its own numbered migration.
- Pre-production, migrations carry schema only, never data fixes. A column drop or rename happens through a rebaseline the owner decides; dev databases are recreated afterwards.
- Keep one golden copy of the complete current schema (tables and indexes only) beside the migrations. Update it in the same commit as every migration and add a drift check.
- Manual mode: apply with a fail-fast flag (`psql "$DATABASE_URL" -v ON_ERROR_STOP=1 -f ...`, since plain `psql -f` exits 0 on failed statements). DDL runs as the table owner; the runtime role gets DML grants only. Confirm destructive changes with the owner.

## Transactions

- One transaction does not make two reads agree under read-committed isolation (each statement has its own snapshot). When you need "stored but unreadable" versus "absent", get both from one statement.
- A "write only if changed" helper must not return null: the write has committed by then, so null turns a committed change into a server error. Return the row or a result object.
- Keep transactions short and database-only; no external calls inside.

## Mappers (DB to API)

Put mappers in `lib/mappers.js`, named `mapXToY`. Return `null` for null input. Use boolean presence fields from companion rows (`undefined` omits them from JSON). Keep hashes and internal flags out of responses.

## Common Mistakes

- Text enums or statuses; status values off the shared convention.
- Hardcoded magic numbers.
- Upserts or `ON CONFLICT` standing in for idempotency; unapproved unique indexes.
- CHECK, DEFAULT (other than `created_at`), triggers or FKs added unprompted.
- Single-column index duplicating a composite's leading column; indexes with no named query.
- Queryable fields inside JSON.
- Check-then-insert with no lock or guarded update.
- Returning raw rows instead of mapping them.
- Assuming the migration mode.
