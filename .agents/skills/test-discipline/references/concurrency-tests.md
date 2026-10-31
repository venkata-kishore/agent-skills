# Concurrency tests

## Racing two requests

- Order the two sides deterministically: hold the contended row, queue the first request, release, then run the second.
- Pin each side's exact answer.
- Verify which side loses before asserting. Sometimes the loser legitimately succeeds and is reconciled later; a brief that expects an error the route never returns makes a wrong test.

## Helpers: fail fast, scoped, bounded

- A wait for a blocked writer counts only waiters behind the holder's own connection. An unscoped count includes other tests' waiters.
- The wait fails at once if the writer throws, or if it returned without blocking.
- Every latch, future and join has a bound. An unbounded wait can hold a shared lock until the module timeout.
- A child-process helper drains the child's output on its own thread before the bounded wait, and kills the child on timeout.

## Folding helper copies into one shared helper

- Check each caller's meaning, not only that it compiles. A private copy can bind a hidden default (the tenant, the clock, the connection pool). Example: a fold exposed a helper that counted one tenant's rows while the test acted in another.
- Size the fold from a scan of the actual call shapes, not from a reviewer's count.
- Propose the full helper API with counts first, then fold.
