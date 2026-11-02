# Gates on a shared machine

Two builds in one checkout clobber each other (a false red), and several test runs across
worktrees can exhaust memory.

## Lock
Every run that executes tests takes one machine-wide lock (`flock`). Compile-only runs need none.
A script that holds the lock through an open file descriptor must close it for child processes
(in bash, `9>&-` on the command). Otherwise a build daemon the run starts inherits the lock and
keeps it after the run ends.

Exit codes, targeted runs and test scope: see `writing-tests`.

## Batched runs
- A worker checks a change with a compile (no lock) and the tests it touches, then commits.
- The lead merges several ready branches and runs one full gate for the batch, then sends each
  failure to the stream best placed to diagnose it (regression or flake, and whose change).
- Do not block on test runs for trivial changes. Queueing dozens of builds on one lock stalls everyone.

## Shared checkout
A gate in a shared checkout also compiles other streams' uncommitted edits. Before committing,
confirm a pass or failure is not caused by files outside your claim.
