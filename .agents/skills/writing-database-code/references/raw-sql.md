# Raw SQL with Drizzle

Open when the builder cannot express a query, or when hand-writing SQL.

Fall back to `db.execute(sql`...`)` only for CTEs, window functions, `generate_series`, complex aggregations and dynamic multi-table queries. With postgres-js, `db.execute` returns the row array directly (a `RowList`), not `{ rows }`.

```js
import { sql } from "drizzle-orm";

export async function getMonthlyStats(userId) {
  const rows = await db.execute(sql`
    SELECT TO_CHAR(created_at, 'YYYY-MM') AS month, COUNT(*)::int AS total
    FROM things
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

JSONB merge. Pass the object as the parameter: since rc.5 the postgres-js driver serialises JSON itself, so a manual `JSON.stringify` double-stringifies. For an array value use `sql.param(value)`.

```js
details: sql`COALESCE(${table.details}, '{}'::jsonb) || ${JSON.stringify(update)}::jsonb`
```

Partial-update guard for PATCH endpoints. The driver takes a JS `Date`, so convert a Luxon value (the date library rule is in the stack skills):

```js
import { DateTime } from "luxon";

export async function updateFields(id, fields) {
  const updates = {};
  if (fields.name !== undefined) updates.name = fields.name;
  if (fields.email !== undefined) updates.email = fields.email;
  if (Object.keys(updates).length === 0) return null;
  const [row] = await db
    .update(table)
    .set({ ...updates, updatedAt: DateTime.utc().toJSDate() })
    .where(eq(table.id, id))
    .returning();
  return row;
}
```

## Style

- UPPERCASE keywords (`SELECT`, `FROM`, `WHERE`); `snake_case` identifiers so you never need to quote them; quote only reserved words or special characters.
- Multi-line and readable, one clause per line; no run-on single lines.
- Short, meaningful table aliases (`orders AS o`, not `x` or `temp`); always use `AS` for column aliases.
- `column IS NULL` / `IS NOT NULL`, never `= NULL`.
- `WHERE` filters before aggregation; `HAVING` filters after.
- `COALESCE(col, default)` for defaults instead of a `CASE`; format any `CASE` one branch per line, and prefer a CTE over deeply nested `CASE`.
- Break long queries into `WITH` CTEs; comment non-obvious logic inline.
