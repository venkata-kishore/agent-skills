# Reviewing a change

Open when reviewing a diff (your own before a PR, a helper's, or someone else's), when reporting
findings, or when answering them.

## Who reviews

- A reviewer with a fresh context: a helper agent or a separate session that did not write the
  change. The session that wrote it has already accepted its own reasoning. Where the agent has a
  built-in review command that runs in a helper, that counts.
- Give the reviewer the diff, what the change is meant to do, and the repo's conventions. Do not
  give it your conclusion: a reviewer handed "this is correct" tends to agree.
- Before a PR that changes code is opened, the diff gets one such review. A change to docs or
  other text only needs none.

## What the reviewer checks

- Does the change do what it says it does?
- Does it fit the code around it, and the repo's conventions?
- Does it break something else: a caller, a second writer of the same data, an error path?
- For a bug fix: does it fix the cause, or only the case that was reported?

Correctness comes first. What a formatter, a linter or a type checker already enforces is not a
finding.

## Reporting findings

- **Evidence.** Every finding cites a file and line and says which input or state leads to which
  wrong result. Read the code path before claiming it; a guess from a name is not a finding.
- **Labels.** Each finding carries one:

  | Label | Meaning |
  |---|---|
  | Important | Breaks behaviour, loses or leaks data, or blocks undoing the change. Fix before merge. |
  | Nit | Minor. At most five per review. |
  | Pre-existing | A real problem the change did not introduce. Listed apart, so it does not read as the author's mistake. |

- **The class, not the line.** When the same mistake appears in several places, report it once as
  a class and list the places, so the author fixes every one.
- **The summary starts with a count** per label, most severe first. When nothing needs fixing,
  say so in plain words.

## Answering findings

- Every finding gets an answer: fixed, with where; or closed with a reason. The reasons are
  "wrong" with the evidence, "intentional" with why, or "not in this change" with where it is
  recorded instead.
- Fix the whole class a finding names, not only the line it cites.
- A pre-existing problem does not widen the change. Record it where the project keeps open work
  and tell the user.

## Rounds

- After fixes, the next round reviews only the new diff, not the whole change again.
- Never repeat a finding that was already reported or closed with a reason. A reviewer that
  disagrees with a closure says so once, with evidence, and leaves the decision to the user.
- Three rounds at most. If the third still finds Important problems, stop fixing line by line:
  step back, rethink the approach and tell the user. A reviewer and an author that are both
  agents can otherwise keep each other busy without the change getting better.
