---
name: writing-tests
description: "Sets test discipline: scope and levels, fixtures, hand-written fakes, injected clocks, timeouts, concurrency tests, structural guard scripts, and reading results after a targeted run. Has stack notes for the JVM, PostgreSQL lock waits and workflow test servers with time skipping. Use when writing, changing, reviewing or debugging tests in any language, or when the task mentions mock, mocking, coverage, snapshot, Temporal, fixtures, flaky CI, a hanging or slow test, a race, or '0 tests ran'."
---

# Test discipline

A project's own conventions override this skill.

## Scope

- Write unit tests on functions and methods by default.
- For integration, component or end-to-end tests, check the project's conventions first. If they do not say, ask the user which levels to write before writing any.
- Sensible tests only: workflows, edge cases, positive and negative paths. No padding.
- A test that asserts a constant equals the same literal written in the test is padding. Instead, move the decision (for example, mapping an error code to a message and a next step) into a pure helper and test the helper.
- When a test needs a constant that is not accessible from the test (for example a limit declared private to its package or module), widen its visibility so the test can import it. Do not copy its number into the test; the test then follows the source.
- During a large structural refactor, if the user agrees, defer test writing: check with compile-only runs, then do a planned test pass on its own branch.

## Fakes and fixtures

- Write fakes by hand. Do not use a mocking library.
- Drive the real production path through one shared fixture. A test-local copy of the path drifts silently (a private "submit" helper can drop production steps).
- A test that calls an internal seam directly does not pin the production wiring: revert the call site and it still passes. Drive the real path with a fake transport that fails deterministically.
- A test that deliberately creates data that breaks a database-wide invariant check (for example a row with no parent) deletes that data when it finishes, so later checks do not fail.
- Test classes that share one database each use their own range of ids (for example 1000-1999 for one class, 2000-2999 for the next) so they cannot collide.

## Output, time and assertions

- Test output is pristine: a warning in the output is a finding. A test that triggers a log line on purpose captures and asserts it. Never lower the root log level to hide it.
- Time comes from an injected clock, never the wall clock. Assert on recorded events (for example, that a timer-started event appears in the workflow history) rather than elapsed time, which drifts under time-skipping servers.
- Two timers with the same budget race: when a library's request timeout and your own deadline are equal, either error can surface. Make the test assert one fixed message for every timeout path, and give the two timers different budgets.
- To check that an error does not echo input, assert on the specific response field, not the whole body. Random ids in the body can contain the probe.
- A rename of how a log line names something can change how many lines a throttle lets through. Say so in the change and update the tests that count lines.

## Timeouts, hangs and flakes

- Every test module sets a default per-test timeout, so one hung test cannot hold a shared lock for everyone. Know what the timeout can stop: see the stack notes below.
- Prove the fix for a hang or flake with a mutation: remove the guard the test exists for and watch it fail, restore it and watch it pass.
- For racing requests, helper waits and child processes, read `references/concurrency-tests.md`.

## Running tests

- After a targeted run, check that the result files list the classes you meant. A test-name filter that matches nothing passes silently.
- A change to a shared fixture (base class, seed helper, race helper) means the whole module's tests, not only the edited classes: every caller depends on it.
- Never pipe a gate's exit code away. Redirect output to a file and print `EXIT=$?` explicitly. Gate on the command's own exit code.
- Test runs that share a machine follow the project's lock rule.

For layering or cycle guard scripts and their tests, read `references/structural-guards.md`.

## Stack notes (open only for that stack)

| Open | When |
|---|---|
| `references/jvm.md` | JVM test timeouts, interrupts, shared containers |
| `references/postgresql.md` | tests that wait on database locks |
| `references/temporal-test-server.md` | workflow tests on a time-skipping test server |
