---
name: writing-database-code
description: "Applies the user's house rules for the data layer of a SQL database (PostgreSQL, Drizzle ORM): no foreign keys, no CHECK, DEFAULT or triggers, integer status values, unique-index approval, idempotent writes, short DB-only transactions, mappers. Use when writing or reviewing database code, Drizzle queries or schema, migrations, indexes, unique constraints, status or enum columns, ID generation, JSON columns, raw SQL, shared constants or DB-to-API mappers. These rules are defaults unless the user says otherwise."
---

# Writing database code

Data-layer rules for any SQL database; PostgreSQL is assumed where it matters.

## References

- `references/drizzle.md`: open when the project uses Drizzle ORM (query builder, relations v2, client wiring, transactions, atomic claims, key rules).
- `references/raw-sql.md`: open when the Drizzle builder cannot express a query (CTEs, window functions, dynamic filters, JSONB merge) or when hand-writing SQL.
- `references/migrations.md`: open when adding or changing a migration or schema file.
- `references/ids-and-constants.md`: open when generating ids, defining shared constants or status objects, or writing DB-to-API mappers.
- `references/postgres.md`: open when creating a PostgreSQL database or cluster, choosing collation, or reasoning about transaction isolation.

## Organization

One file per entity in a shared queries directory (for example `<entity>.queries.js`), each exporting named async functions. Feature modules keep route, service and schema files together; database access goes in the shared query files.

## Schema

- **Known-value fields are small-integer codes, never text.** Use the smallest type that fits (`smallint`), numbered without gaps. Emit them to the frontend as generated constants.
- **Bigint ids everywhere**, serialised as strings for JavaScript (bigint exceeds JS number precision). Counts and other numeric columns may be right-sized.
- **Keep the schema simple.** Fewer tables; a column instead of a side table where queries allow. Split only when a query needs it. Separate preferences (visible to everyone) from settings (access-controlled).
- **Small static sets live in code** as shared constants, not a lookup table, until the set grows or changes at runtime.
- **No foreign keys by default.** The app layer owns referential integrity and writes related rows in one transaction. Add FKs only when the user asks.
- **No CHECK constraints, no DEFAULT clauses except `created_at`, no triggers.** Rules live in code with tests. An exception needs the user's explicit approval.
- **Real columns for anything queried.** Filter, join, sort or index fields are columns, not JSON keys.
- **JSON columns** (`jsonb`) are nullable with no default and read and written as a whole. Stored keys are short, declared as constants, with a comment explaining the abbreviations. API bodies keep full names.
- **Canonicalise at write** (lower-case emails and hostnames) so lookups use plain equality and plain indexes, not case-insensitive collations or `lower()` in queries.
- **Text limits** count characters: 2,000 by default, smaller for short fields, 5,000 for bodies, unless decided otherwise.
- **Time is UTC.** Servers and database run in UTC; set the process default zone at boot before anything else initialises.
- **Sort order** for typed names is deterministic: code-point order unless the user asks for linguistic order. Record the choice in the project's conventions file.
- **Archive instead of deleting reference data.** Archiving blocks new use; in-flight work runs to completion.

## Indexes and uniqueness

- No index until a named query needs it. Name that query when you add the index.
- Prefer one composite index over a single-column index with the same leading column.
- Every new unique index needs the user's approval, including event-dedup tables.
- A unique index is an integrity constraint, never the basis of an upsert. Do not use `ON CONFLICT` to get idempotency.

## Idempotent writes

Every job and write path is safe to repeat: separate a side effect from recording it, skip settled rows, re-check the condition inside the guarded `UPDATE`, treat an already-missing object as done.

Check-then-insert is not safe alone: two concurrent transactions can both see nothing and both insert. Serialise first, either by locking the owning row (a guarded update or `SELECT ... FOR UPDATE`) before the check, or by claiming with a conditional `UPDATE ... WHERE status = <expected>` and treating a returned row as a won claim. Drizzle examples in `references/drizzle.md`.

## Tenancy

Anything that runs outside a web request (jobs, workflows, queues, object keys, exports, log lines) carries the tenant id with every entity id. Never encode a location (shard, region, database) in an id. Worker ids are unique across every running instance.

Tenancy without row-level security: one scoped data-access gateway, and every mutation writes exactly one audit row in the same transaction.

## Row status values

One convention across every table (documented exceptions only):

| Value | Meaning |
|------|---------|
| `0` | base marker (row exists, nothing else known) |
| `1` | draft |
| `2` | ended normally |
| `3` | archived |
| `> 3` | live states, ordered by liveness |
| `< 0` | removable without real loss |

Range checks work directly: `status > 3` is live, `status < 0` is removable, `ORDER BY status` follows the lifecycle.

## Constants, ids, mappers

- Constants live in a shared package imported by backend and frontend. Never hardcode magic numbers.
- Choose ids by scale: snowflake-style `bigint` for high-traffic tables, identity columns for small reference tables. Pre-generate ids when several tables in one transaction must reference them.
- Mappers are named `mapXToY`, return `null` for null input, and keep hashes and internal flags out of responses.
- Examples and detail: `references/ids-and-constants.md`.

## Migrations

Determine the mode first (migrations directory, ORM config, package scripts): tool-managed (edit the schema, run the generator, never hand-edit generated SQL) or manual SQL (numbered files plus the schema file, applied by a human). Write idempotent DDL. Pre-production, migrations carry schema only. Details: `references/migrations.md`.

## Transactions

- One transaction does not make two reads agree under read-committed isolation (each statement has its own snapshot). When you need "stored but unreadable" versus "absent", get both from one statement.
- A "write only if changed" helper must not return null: the write has committed by then, so null turns a committed change into a server error. Return the row or a result object.
- Keep transactions short and database-only; no external calls inside (see `securing-applications` for outbound timeouts).
