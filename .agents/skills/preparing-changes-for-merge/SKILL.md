---
name: preparing-changes-for-merge
description: "Guides the steps between a finished change and a merged one: reviewing a diff with a fresh-context reviewer, reporting and answering review findings, capping review rounds, and writing commit messages, PR titles and PR descriptions. Use before every git commit, gh pr create or gh pr edit, when amending or rewording a message, when a PR's scope changes after it was opened, when reviewing a diff or a pull request, when asked for a code review, or when answering review findings."
---

# Preparing a change for merge

**Scope.** What happens to a change between "the code is written" and "it is merged": the
review, the answers to its findings, the commit messages, and the PR title and description.
Running the checks themselves is `writing-tests`.

## The order

1. **Check.** Run the repo's gates and keep their output. Only checks that really ran go into
   the text later.
2. **Review.** Before opening a PR that changes code, a reviewer with a fresh context reviews the
   diff. A change to docs or other text only needs no review.
3. **Answer.** Every finding is fixed or closed with a reason. Nothing is left silent.
4. **Write.** Commit messages and the PR text say what changes, why, and how it was checked.
5. **Open the PR** only as the user's rules for outward-facing actions allow.

## References

Open the one for the step you are on. Each holds the rules for that step in full.

| Open | When |
|---|---|
| `references/reviewing.md` | reviewing a diff, reporting findings, answering findings, a second or third review round |
| `references/commit-and-pr-text.md` | before writing or editing a commit message, a PR title or a PR description |
