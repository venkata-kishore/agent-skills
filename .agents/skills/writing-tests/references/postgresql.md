# PostgreSQL lock waits in tests

- `lock_timeout` aborts a statement that waits longer than the limit for a lock on a table, index, row or other object, explicit or implicit. A bare number is milliseconds; 0 (default) disables it.
- Set it on the test database or session, not in `postgresql.conf` for a shared server: the docs advise against setting it there because it affects all sessions.
- Keep it well under the test timeout, and below any `statement_timeout` (a statement timeout that is equal or smaller fires first and makes `lock_timeout` pointless).
- Use it because a client-side interrupt cannot end a blocked driver read (see `jvm.md`).
