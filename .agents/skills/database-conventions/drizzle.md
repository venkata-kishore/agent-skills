# Drizzle ORM Query Patterns

Query mechanics for the data layer. See `SKILL.md` for file organization, status conventions, constants, ID strategy, and mappers.

## Default: Use Drizzle's Standard API

Leverage Drizzle's query builder as the primary pattern — express everything you can through the ORM and drop to raw SQL only when the builder genuinely cannot do it. It handles selects, inserts, updates, deletes, joins, filtering, ordering, and pagination natively.

```js
import { eq, and, gt, sql } from "drizzle-orm";
import { db } from "../client.js";
import { things } from "../schema.js";

export async function findById(id) {
  const [row] = await db.select().from(things).where(eq(things.id, id)).limit(1);
  return row ?? null;
}

export async function listByStatus(status, { limit, offset }) {
  return db
    .select()
    .from(things)
    .where(eq(things.status, status))
    .orderBy(sql`${things.createdAt} DESC`)
    .limit(limit)
    .offset(offset);
}

export async function create(data) {
  const [row] = await db.insert(things).values(data).returning();
  return row;
}

export async function updateStatus(id, newStatus) {
  const [row] = await db
    .update(things)
    .set({ status: newStatus, updatedAt: new Date() })
    .where(eq(things.id, id))
    .returning();
  return row;
}

export async function remove(id) {
  const [row] = await db.delete(things).where(eq(things.id, id)).returning();
  return row;
}

export async function findActive(orgId, afterId) {
  return db
    .select()
    .from(things)
    .where(and(eq(things.orgId, orgId), eq(things.status, ACTIVE), gt(things.id, afterId)))
    .limit(50);
}
```

## Relations & Relational Queries (v2)

Drizzle's relations and relational query builder are now v2 (shipped with Drizzle ORM v1). Use the v2 conventions. The core `db.select()` builder above is unchanged across versions — this only affects how relations are defined and how related rows are read via `db.query`.

Define all relations in one place with `defineRelations` (replaces the old per-table `relations()`); fields are `from`/`to` (were `fields`/`references`), and the client takes `{ relations }` (was `{ schema }`):

```js
import { defineRelations } from "drizzle-orm";
import * as schema from "./schema.js";

export const relations = defineRelations(schema, (r) => ({
  users: {
    invitee: r.one.users({ from: r.users.invitedBy, to: r.users.id }),
    posts: r.many.posts({ from: r.users.id, to: r.posts.authorId }),
  },
}));

const db = drizzle(url, { relations });
```

Read related rows with `db.query` using object syntax — not the v1 callback form:

```js
// v2
db.query.users.findMany({ where: { id: 1 }, orderBy: { id: "asc" }, with: { posts: true } });

// v1 (deprecated): where: (u, { eq }) => eq(u.id, 1), orderBy: (u, { asc }) => [asc(u.id)]
```

Many-to-many uses `.through()` on the junction columns, then `with: { groups: true }` in the query.

Reference: [relations v1→v2](https://orm.drizzle.team/docs/relations-v1-v2) and the [v1 beta release notes](https://orm.drizzle.team/docs/latest-releases/drizzle-orm-v1beta2) (which also move RLS to `pgTable.withRLS(...)` and restructure the migrations folder via `drizzle-kit up`).

## Client Wiring (v1 Gotcha)

In v1 (1.0.0-rc and later), `drizzle()` treats its first argument as a config object unless it is a connection string. A pre-built driver client MUST be passed as `{ client }`:

```js
const sql = postgres(env.DATABASE_URL, { prepare: false });

const db = drizzle({ client: sql, relations }); // correct
const db = drizzle(url, { relations });         // also fine — a string is detected as a URL
const db = drizzle(sql, { relations });         // TRAP — see below
```

The legacy positional form `drizzle(sql, { relations })` fails silently: the client instance is destructured as if it were config, `client`/`connection` come out `undefined`, and drizzle constructs a brand-new default client pointing at **localhost:5432** — ignoring both your real client and your `relations`.

Diagnostic signature: every runtime query fails with `CONNECT_TIMEOUT ["localhost"]:5432` while the env URL is correct, a raw driver client connects fine, and migrations succeed (migrate scripts are often already on the `{ client }` form — that asymmetry is the tell). Do not chase network, DNS, or env loading — inspect the `drizzle()` call site first and confirm with `db.$client === sql` (or check `db.$client.options.host`).

## Transactions

Optional transaction parameter pattern — every query function that may participate in a transaction accepts `tx = db`:

```js
export async function createItem(data, tx = db) {
  const [row] = await tx.insert(items).values(data).returning();
  return row;
}

await db.transaction(async (tx) => {
  const item = await createItem(data, tx);
  await linkToUser(item.id, userId, tx);
  await incrementCounter(userId, tx);
});
```

## Idempotency & Atomic Claims

App-layer guarantees, per the no-FK / no-unique-index policy in `SKILL.md`.

Check-then-insert inside a transaction for business idempotency:

```js
await db.transaction(async (tx) => {
  const existing = await findByUserAndPlan(userId, planId, tx);
  if (existing) return existing;
  return await createSubscription({ userId, planId }, tx);
});
```

Atomic claim via conditional update so two workers can't pick the same row — a returned row means the claim was won:

```js
const [claimed] = await tx
  .update(jobs)
  .set({ status: JOB_STATUS.PROCESSING })
  .where(and(eq(jobs.id, id), eq(jobs.status, JOB_STATUS.PENDING)))
  .returning();
if (!claimed) return; // another worker won
```

Event-dedup inserts — the one sanctioned unique target (e.g. webhook events keyed by provider + event id):

```js
await db.insert(events).values(evt).onConflictDoNothing({ target: [events.provider, events.eventId] });
```

## Complex Queries (Raw SQL)

Fall back to `db.execute(sql`...`)` only when Drizzle's builder is insufficient — CTEs, window functions, `generate_series`, complex aggregations, dynamic multi-table queries:

```js
export async function getMonthlyStats(userId) {
  const { rows } = await db.execute(sql`
    SELECT TO_CHAR(created_at, 'YYYY-MM') AS month, COUNT(*)::int AS total
    FROM app.things
    WHERE user_id = ${userId}
    GROUP BY month
    ORDER BY month DESC
  `);
  return rows;
}
```

Dynamic `WHERE` builder for admin list endpoints with optional filters:

```js
const conditions = [];
if (status != null) conditions.push(sql`t.status = ${status}`);
if (search) conditions.push(sql`t.name ILIKE ${"%" + search + "%"}`);
const where = conditions.length > 0 ? sql`WHERE ${sql.join(conditions, sql` AND `)}` : sql``;
```

JSONB merge:

```js
details: sql`COALESCE(${table.details}, '{}'::jsonb) || ${JSON.stringify(update)}::jsonb`
```

Partial-update guard for PATCH endpoints:

```js
export async function updateFields(id, fields) {
  const updates = {};
  if (fields.name !== undefined) updates.name = fields.name;
  if (fields.email !== undefined) updates.email = fields.email;
  if (Object.keys(updates).length === 0) return null;
  const [row] = await db
    .update(table)
    .set({ ...updates, updatedAt: new Date() })
    .where(eq(table.id, id))
    .returning();
  return row;
}
```

### Raw SQL Style

When the builder is insufficient and you hand-write SQL, keep it consistent and bug-resistant:

- UPPERCASE keywords (`SELECT`, `FROM`, `WHERE`); `snake_case` identifiers so you never need to quote them — quote only reserved words or special characters.
- Keep it multi-line and readable, one clause per line; no run-on single lines.
- Short, meaningful table aliases (`orders AS o`, not `x` or `temp`); always use `AS` for column aliases.
- `column IS NULL` / `IS NOT NULL` — never `= NULL`.
- `WHERE` filters before aggregation; `HAVING` filters after.
- `COALESCE(col, default)` for defaults instead of a `CASE`; format any `CASE` one branch per line, and prefer a CTE over deeply nested `CASE`.
- Break long queries into `WITH` CTEs; comment non-obvious logic inline.

## Key Rules

- v1 `drizzle()` takes a config object: pass a pre-built client as `drizzle({ client })`. Positional `drizzle(client, ...)` silently connects to localhost:5432 (see Client Wiring).
- Always use `.returning()` on insert/update/delete.
- Single-record queries destructure `const [row]` and return `row ?? null`.
- Keep transactions short and DB-only — no external API calls inside.
- Pass `tx` through the entire call chain for atomicity.
- BigInt columns use `{ mode: "string" }`; handle IDs and large integers as strings.
- Use Luxon `DateTime` for date logic, comparisons, and formatting; avoid native `Date` for business logic.
- Add `.orderBy(...)` whenever order matters — SQL guarantees no row order otherwise, and stable pagination depends on it.
- Test existence with a `LIMIT 1` row check (`const [row] = await db.select(...).limit(1); return row != null`), never `COUNT(*)`.
- On wide or hot-path tables, select explicit columns (`db.select({ id: t.id, name: t.name })`) instead of relying on `db.select()` plus a mapper to drop fields.
