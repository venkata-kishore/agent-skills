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

### Why integers

Ids are `bigint` (8 bytes), not UUIDs (16 bytes). Indexes and foreign keys stay half the size, joins compare one machine word, and time-ordered ids append at the right edge of the btree instead of splitting pages at random, as UUIDv4 does. Small reference tables may use identity columns.

### Layout

A signed 64-bit value that is always positive:

| Bits | Field |
|---|---|
| 1 | sign, always 0 |
| 41 | milliseconds since a custom epoch (a fixed constant per project, for example 2026-01-01T00:00:00Z; never change it). 2^41 ms is about 69.7 years |
| 22 | low bits, filled by the mode below |

Combine with OR, never AND: AND with a value shifted left by 22 clears the low bits, so every id in a millisecond would be the same.

```text
id = ((millis - EPOCH) << 22) | low
```

### Modes

Pick one mode per deployment at boot, from the software's needs and what the environment has. Every process of a deployment uses the same mode: ids minted in different modes can collide in the same millisecond, so a process never falls back to another mode while running.

1. Central counter (Redis if available, else a Postgres sequence): `low = counter % 4194304` (2^22), where the counter is Redis `INCR` on one key or Postgres `nextval` on one `bigint` sequence. Use when the system needs one source handing out numbers.
   - The mod keeps `low` inside 22 bits however large the counter grows, so it never spills into the timestamp bits. For a non-negative counter it equals `counter & 4194303`.
   - No worker id, and a restart cannot repeat ids because the counter lives outside the process.
   - Two ids collide only if the counter advances 2^22 times while hosts label ids with the same millisecond (one millisecond, plus any clock skew between hosts). The mod keeps the value in range; it does not prevent that wrap.
   - Every id costs a round trip. Take a block at a time (`INCRBY n`, or a sequence with `CACHE n`) when that is too slow.
2. Leased worker id: Redis or Postgres only hands each process a worker id at boot; minting is local, as in mode 3. Use when no central numbering is needed: no round trip per id, and no hand-set `WORKER_ID` to get wrong.
   - The lease has an expiry and is renewed well before it. A process that fails to renew stops minting before the lease expires, otherwise another process can take the same worker id and both mint the same ids.
3. Local: `low = (workerId << 12) | counter`, with a 10-bit worker id (0 to 1023) and a 12-bit per-millisecond counter (0 to 4095). The worker id comes from configuration.

### Local minting rules (modes 2 and 3)

- In mode 3 the worker id comes from configuration (for example `WORKER_ID`), validated at boot; a bad value fails boot naming only the variable. It must be unique across every running instance in every region. Nothing detects a duplicate, and two generators with the same worker id mint the same ids.
- Exactly one generator per process, built at boot and injected everywhere (`building-backends`). Test harnesses share one generator per test JVM or process, not one per test.
- Timestamp, counter and the last minted millisecond move together under one lock (or one atomic value advanced by compare-and-set).
- Put the counter behind one small interface (given the millisecond, return 0 to 4095 or exhausted), so a central source can replace the local one without touching the minting code.
- Counter exhausted within a millisecond: move the logical timestamp forward 1 ms and ask for a fresh counter there; never wrap to 0 in the same millisecond (0 with the stale millisecond repeats an id), and never wait for the clock. A second exhaustion on the fresh millisecond is a broken source: throw. Under sustained load the embedded time runs ahead of the real clock by up to 1 ms per 4,096 ids.
- Clock steps back: mint at max(clock, last minted millisecond), so ids stay unique and increasing. Log one warning (the step size only) for a large step.
- A reading before the epoch, or past the 41-bit range, throws and mints nothing. Check the range before touching the counter, and again for the +1 ms retry: the counter never moves its window back, so a refused future millisecond would keep it counting there.
- Known risk: the counter and last-millisecond memory live in memory. A restart into a clock behind the previous run's last minted millisecond, on the same worker id, can repeat ids.

### Use

- Pre-generate ids before insert when several tables in one transaction must reference them.
- Ids never encode a shard, region or database.
- Ids travel to clients as strings: 64-bit values lose precision in JavaScript numbers.

## Mappers (DB to API)

Put mappers in one shared module (for example `lib/mappers.js`), named `mapXToY`. Return `null` for null input. Keep hashes and internal flags out of responses.

Boolean presence fields come from companion rows. Pass `undefined` when the companion was not loaded, so JSON omits the field:

```js
export const mapUserToApi = (row, avatar) =>
  row && { id: String(row.id), name: row.name, hasAvatar: avatar === undefined ? undefined : avatar !== null };
```
