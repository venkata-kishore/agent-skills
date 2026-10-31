---
name: writing-tests
description: "Use when writing, changing, reviewing or debugging tests, in any language: choosing test scope and levels, fixtures, fakes, injected clocks, test timeouts, flaky, hanging or slow tests, race and concurrency tests, structural guard scripts, and reading test results after a targeted run. Has stack notes for the JVM, PostgreSQL lock waits and workflow test servers with time skipping."
---

# Test discipline

## Scope

- Write unit tests on functions and methods by default.
- For integration, component or end-to-end tests, check the project's conventions first. If they do not say, ask the owner which levels to write before writing any.
- Sensible tests only: workflows, edge cases, positive and negative paths. No padding.
- A test that compares a constant with its own literal is padding. Move the decision (code to message, plus what to do next) into a pure helper and test the helper.
- When a test needs a constant it cannot see (a package-private bound), make the constant visible. Do not repeat its number in the test; the test then follows the source.
- During a large structural refactor, defer test writing: check with compile-only runs, then do a planned test pass on its own branch.

## Fakes and fixtures

- Write fakes by hand. Do not use a mocking library.
- Drive the real production path through one shared fixture. A test-local copy of the path drifts silently (a private "submit" helper can drop production steps).
- A test that calls an internal seam directly does not pin the production wiring: revert the call site and it still passes. Drive the real path with a fake transport that fails deterministically.
- A test that leaves rows a global invariant check would see (an orphan row made on purpose) deletes them when it finishes.
- Test classes that share one database use disjoint id ranges so they cannot collide.

## Output, time and assertions

- Test output is pristine: a warning in the output is a finding. A test that triggers a log line on purpose captures and asserts it. Never lower the root log level to hide it.
- Time comes from an injected clock, never the wall clock. Assert on recorded events (a timer-started event) rather than elapsed time, which drifts under time-skipping servers.
- Two timers with the same budget race: when a library's request timeout and your own deadline are equal, either message can surface. Map every timeout to one fixed message.
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

## Structural guards

- A guard (layering, cycles) has no allowed list of known breaks. Break the known violations first, then make the rule strict. An allowed list hides growth.
- Each guard script has its own small test script: throwaway source trees, exact exit code and failure lines per case, plus a mutation check.

## Stack notes (open only for that stack)

| Open | When |
|---|---|
| `references/jvm.md` | JVM test timeouts, interrupts, shared containers |
| `references/postgresql.md` | tests that wait on database locks |
| `references/temporal-test-server.md` | workflow tests on a time-skipping test server |
