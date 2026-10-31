# Migrations

Open when adding or changing a migration or schema file.

Determine the mode first (migrations directory, ORM config, package scripts).

| Mode | Source of truth | Workflow |
|------|-----------------|----------|
| Tool-managed | ORM schema file | Edit schema, run the generator, tooling applies; never hand-edit generated SQL |
| Manual SQL | Numbered SQL files | Write `00N_name.sql` and update the schema file in the same change; a human applies it |

- Write idempotent DDL (`IF NOT EXISTS` / `IF EXISTS`, guarded seed inserts). Seed data lives in its own numbered migration.
- Pre-production, migrations carry schema only, never data fixes. A column drop or rename happens through a rebaseline the user decides; dev databases are recreated afterwards.
- Keep one golden copy of the complete current schema (tables and indexes only) beside the migrations. Update it in the same commit as every migration and add a drift check.
- Manual mode: apply with a fail-fast flag (`psql "$DATABASE_URL" -v ON_ERROR_STOP=1 -f ...`, since plain `psql -f` exits 0 on failed statements). DDL runs as the table owner; the runtime role gets DML grants only. Confirm destructive changes with the user.
