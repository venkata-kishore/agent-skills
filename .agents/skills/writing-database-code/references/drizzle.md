# Drizzle ORM query patterns

Checked against Drizzle ORM 1.0.0-rc.5 ([release page](https://github.com/drizzle-team/drizzle-orm/releases/tag/v1.0.0-rc.5)). Query mechanics for the data layer. See `SKILL.md` for the schema, index, status, constants, ID and mapper rules; raw SQL is in `references/raw-sql.md`.

## Default: use Drizzle's standard API

Use Drizzle's query builder as the primary pattern: express everything you can through the ORM and drop to raw SQL only when the builder genuinely cannot do it. It handles selects, inserts, updates, deletes, joins, filtering, ordering, and pagination natively.

```js
import { eq, and, gt, sql } from "drizzle-orm";
import { DateTime } from "luxon";
import { db } from "../client.js";
import { things } from "../schema.js";
import { THING_STATUS } from "@app/shared/constants";

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

// The driver takes a JS Date, so convert the Luxon value at the boundary.
export async function updateStatus(id, newStatus) {
  const [row] = await db
    .update(things)
    .set({ status: newStatus, updatedAt: DateTime.utc().toJSDate() })
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
    .where(and(eq(things.orgId, orgId), eq(things.status, THING_STATUS.QUEUED), gt(things.id, afterId)))
    .limit(50);
}
```

## Relations and relational queries (v2)

Relations and the relational query builder are v2 in Drizzle ORM v1. `db._query` (relational queries v1) is removed for Postgres since rc.1. The `db.select()` builder above is unaffected; this only changes how relations are defined and read via `db.query`.

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

Read related rows with `db.query` using object syntax, not the v1 callback form:

```js
// v2
db.query.users.findMany({ where: { id: 1 }, orderBy: { id: "asc" }, with: { posts: true } });

// v1 (deprecated): where: (u, { eq }) => eq(u.id, 1), orderBy: (u, { asc }) => [asc(u.id)]
```

Many-to-many uses `.through()` on the junction columns, then `with: { groups: true }` in the query.

Since rc.5, `undefined` in a relational-query `where` throws. When a filter may be absent on purpose, use `EmptyFilter`:

```js
import { EmptyFilter } from "drizzle-orm/relations";

db.query.users.findMany({ where: { isActive: params.isActive ?? EmptyFilter } });
```

An empty relation filter (`where: { posts: {} }`) is skipped. Filter by a relation's presence with `posts: true` and its absence with `posts: false`.

Reference: [relations v1 to v2](https://orm.drizzle.team/docs/relations-v1-v2) and the [rc.5 release page](https://github.com/drizzle-team/drizzle-orm/releases/tag/v1.0.0-rc.5).

## Casing and error params

Column casing is set on the table, view or schema, no longer via `drizzle({ casing })` (removed in rc.1):

```js
import * as d from "drizzle-orm/pg-core";

export const things = d.snakeCase.table("things", { createdAt: d.timestamp() }); // created_at
```

`DrizzleQueryError` omits query params by default (rc.5). Keep `paramsInErrors` off: params can carry personal data (see `securing-applications`).

## Client wiring (v1 gotcha)

In v1 (1.0.0-rc and later), `drizzle()` treats its first argument as a config object unless it is a connection string. A pre-built driver client MUST be passed as `{ client }`:

```js
const sql = postgres(env.DATABASE_URL, { prepare: false });

const db = drizzle({ client: sql, relations }); // correct
const db = drizzle(url, { relations });         // also fine: a string is detected as a URL
const db = drizzle(sql, { relations });         // TRAP: see below
```

The legacy positional form `drizzle(sql, { relations })` fails silently: the client instance is destructured as if it were config, `client`/`connection` come out `undefined`, and drizzle constructs a brand-new default client pointing at **localhost:5432**: ignoring both your real client and your `relations`.

Diagnostic signature: every runtime query fails with `CONNECT_TIMEOUT ["localhost"]:5432` while the env URL is correct, a raw driver client connects fine, and migrations succeed (migrate scripts are often already on the `{ client }` form: that asymmetry is the tell). Do not chase network, DNS, or env loading: inspect the `drizzle()` call site first and confirm with `db.$client === sql` (or check `db.$client.options.host`).

## Transactions

Optional transaction parameter pattern: every query function that may participate in a transaction accepts `tx = db`:

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

## Idempotency and atomic claims

App-layer guarantees, per the no-unique-index-as-upsert rule in `SKILL.md`.

A bare check-then-insert is unsafe: two concurrent transactions both find nothing and both insert. Take a row lock on the owning row first, so the second transaction waits and then sees the first insert. A guarded update on the owner holds the lock until commit:

```js
await db.transaction(async (tx) => {
  const [owner] = await tx
    .update(users)
    .set({ updatedAt: DateTime.utc().toJSDate() })
    .where(eq(users.id, userId))
    .returning();
  if (!owner) return null;
  const existing = await findByUserAndPlan(userId, planId, tx);
  if (existing) return existing;
  return await createSubscription({ userId, planId }, tx);
});
```

Atomic claim via conditional update so two workers can't pick the same row. A returned row means the claim was won:

```js
const [claimed] = await tx
  .update(jobs)
  .set({ status: JOB_STATUS.PROCESSING })
  .where(and(eq(jobs.id, id), eq(jobs.status, JOB_STATUS.QUEUED)))
  .returning();
if (!claimed) return; // another worker won
```

Event dedup follows the same two patterns (lock or claim). It needs no unique index unless the user approves one as an integrity constraint; even then the insert is not an upsert.

## Key Rules

- v1 `drizzle()` takes a config object: pass a pre-built client as `drizzle({ client })`. Positional `drizzle(client, ...)` silently connects to localhost:5432 (see Client Wiring).
- Always use `.returning()` on insert/update/delete.
- Single-record queries destructure `const [row]` and return `row ?? null`.
- Keep transactions short and DB-only, no external API calls inside.
- Pass `tx` through the entire call chain for atomicity.
- BigInt columns use `{ mode: "string" }`; handle IDs and large integers as strings.
- Use Luxon `DateTime` for date logic, comparisons and formatting; where the driver takes a JS `Date`, pass `DateTime.utc().toJSDate()`.
- Add `.orderBy(...)` whenever order matters: SQL guarantees no row order otherwise, and stable pagination depends on it.
- Test existence with a `LIMIT 1` row check (`const [row] = await db.select(...).limit(1); return row != null`), never `COUNT(*)`.
- On wide or hot-path tables, select explicit columns (`db.select({ id: t.id, name: t.name })`) instead of relying on `db.select()` plus a mapper to drop fields.
