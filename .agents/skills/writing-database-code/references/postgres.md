# PostgreSQL Specifics

Open when creating a database or cluster, choosing a collation, or reasoning about isolation. Checked against the PostgreSQL 17 and 18 documentation.

## Encoding at creation

Every database is UTF-8 from creation. Pass the encoding and locale explicitly to `initdb` and `CREATE DATABASE`; do not rely on defaults. The `initdb` default template encoding is derived from the locale, and with the `C`/`POSIX` locale and the `libc` provider it is `SQL_ASCII`. Container images can hit this, so a database may be created as `SQL_ASCII` unnoticed. Check with `SHOW server_encoding;`.

## Builtin C.UTF-8 collation

The builtin collation provider was added in PostgreSQL 17 (`C` and `C.UTF-8`; `PG_UNICODE_FAST` is listed in the 18 docs). It does not depend on the host's libc, so it behaves the same on every host. `C.UTF-8` requires UTF-8 and sorts by code point. Prefer it for new clusters on 17 or later.

```sh
initdb --encoding=UTF8 --locale-provider=builtin --builtin-locale=C.UTF-8
```

```sql
CREATE DATABASE app TEMPLATE template0 ENCODING 'UTF8'
  LOCALE_PROVIDER builtin BUILTIN_LOCALE 'C.UTF-8';
```

`template0` is needed because encoding and locale must match the template unless it is `template0`.

Prefix `LIKE 'abc%'` with a plain btree index is documented to work for the `C` locale only; other locales need a `text_pattern_ops` index (and a default-opclass index too if `<`/`>` comparisons must use an index). Do not assume a plain index serves prefix `LIKE` under `C.UTF-8`: check with `EXPLAIN` on the target version.

## Read committed

Under the default read-committed level each statement takes its own snapshot, so two successive statements in one transaction can see different data. Combine reads that must agree into one statement, or use repeatable read.
