# Ids, constants and mappers

Open when generating ids, defining shared constants or status objects, or writing DB-to-API mappers.

## Constants

Constants live in a shared package imported by backend and frontend. Never hardcode magic numbers. Status objects follow the row status convention in `SKILL.md`.

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

## ID generation

Choose by scale. Snowflake-style ids suit high-traffic tables: upper bits are milliseconds since a custom epoch, lower bits a counter, stored as `bigint`. Example counter source: a Redis `INCR` with a local fallback. Small reference tables may use identity columns. Pre-generate ids when several tables in one transaction must reference them.

## Mappers (DB to API)

Put mappers in one shared module (for example `lib/mappers.js`), named `mapXToY`. Return `null` for null input. Keep hashes and internal flags out of responses.

Boolean presence fields come from companion rows. Pass `undefined` when the companion was not loaded, so JSON omits the field:

```js
export const mapUserToApi = (row, avatar) =>
  row && { id: String(row.id), name: row.name, hasAvatar: avatar === undefined ? undefined : avatar !== null };
```
