# Commit messages and PR descriptions

Open before writing or editing a commit message, a PR title or a PR description.

**Scope.** Commit subjects and bodies, PR titles and descriptions, and later edits to either.

## The test

Write for someone reading `git log` or the merged PR a year from now, who has the diff and
nothing else: not this chat, not your plan, not the user's other projects. A sentence earns its
place only if that reader needs it to understand or review this change. Everything else is a
report for the user: say it in chat.

## What belongs

- **What changes**, as behaviour, not as a list of edited lines. The diff already shows the lines.
- **Why**, when the reason is not obvious from the change itself.
- **How it was checked:** the commands really run and their result. Say plainly what failed,
  what was skipped and what was not tested.
- **What a reviewer or upgrader must know:** a behaviour change, a breaking change, a new
  dependency, a new secret or environment variable, a config change, a migration step.
- **How to undo it**, when a plain revert is not enough: a migration that must be reversed, data
  that was rewritten, a flag or setting that must be switched back.

Text and diff match: mention every change of that kind, and describe nothing the diff does not
contain.

## What stays out

- **Anything with no bearing on the change:** release counts, registry or CI quotas, schedules,
  other work in progress, what comes next, notes about the session.
- **Other projects.** Never name another repo or product of the user's, even as context. A
  public link to an upstream bug or spec the change depends on is fine.
- **The journey:** what was tried, how it was debugged, review rounds, "first I", "after
  discussion". Describe the final state.
- **References the reader cannot resolve:** "as discussed", "the above", "the previous commit",
  a chat or session link, a branch that will be deleted, a local or scratch path.
- **Internal ids as a name** ("R5", "item 4", a plan step number) and session, agent or helper
  names. Say what the thing is.
- **Self-attribution:** `Co-Authored-By` trailers, "Generated with ..." lines, tool signatures.
- **Secrets, tokens, authorization codes and personal data.**
- **Words attributed to the user** that they did not write.
- **Unverified claims:** a guessed SHA, version, issue number or date; "tested" when nothing ran.
- **Words that go stale:** "today", "currently", "latest", "new". Give the version or the date.
- **Emojis, em-dashes, openers, praise and filler** ("This PR ...", "simply", "comprehensive").
- **Details of an unreleased security fix** in a public repo. Describe the change neutrally and
  add the detail once the fix is out.

## Title

- It names the change itself, so the reader knows what changed from the title alone. Never the
  activity: "Update files", "Fix bug", "WIP", "Changes", "Address review comments".
- Follow the repo's existing style. Read `git log --oneline -15` first: prefix, tense,
  capitalisation, trailing period or none.
- One commit covers one task. If no single title is true of a commit, split it. A PR that
  carries a batch of changes names each part in its title.
- When later commits change the scope, update the PR title and description in the same step.

## What the text triggers

Check what reacts to the text before writing it:

- **Title patterns a release script or pipeline acts on**, such as a title that starts a release
  or publishes a snapshot. Read the repo's conventions and release docs for them. A wrong title
  releases by accident or fails to release.
- **Conventional-commit types** (`feat`, `fix`, `!`) where a tool derives the version or the
  changelog from them.
- **Issue keywords:** `Fixes #n`, `Closes #n` and `Resolves #n` close the issue on merge, and a
  bare `#n` links it. Check that the number is the one you mean.
- **Mentions:** a bare `@name` notifies that person. Put anything that is not a real mention in
  backticks.
- **Pipeline markers:** `[skip ci]` and the like stop pipelines.

## Squash merges

A squash merge builds the commit on the main branch from the PR, so check which text it takes.
On GitHub:

```bash
gh api repos/{owner}/{repo} --jq '.squash_merge_commit_title, .squash_merge_commit_message'
```

`PR_BODY` means the PR description becomes the commit body. `COMMIT_MESSAGES` means every commit
message on the branch does. Whichever is used is permanent history on the main branch, so write
it to commit standard:

- It reads well as raw text: plain paragraphs and lists, no images, HTML or collapsed sections.
- Notes that matter only until the merge (merge order, a request to a reviewer) go in a PR
  comment, not in the description.

## Getting it through the shell

- Never pass text holding backticks, `$`, quotes or more than one line as an inline `-m "..."`:
  the shell expands or mangles it. Write it to a file (`git commit -F <file>`,
  `gh pr create --body-file <file>`, `gh pr edit <n> --body-file <file>`) or use a quoted
  heredoc (`<<'EOF'`).
- Read the result back: `git log -1 --format=%B`, `gh pr view <n> --json title,body`. Look for a
  literal `\n`, lost blank lines, expanded backticks and leftover template text.
- Before the first commit in a repo, compare `git config user.name` and `git config user.email`
  with the authors already in `git log`. A private email address is personal data.
- When the repo has a PR template, fill every section it requires, delete its placeholder text
  and tick only the boxes that are true.

## Fixing it afterwards

- Not pushed yet: amend the commit.
- Pushed: rewording a commit means a force push, which needs the user's go.
- A PR title or description: edit it in place and tell the user what changed.
- Already merged: it is permanent. Tell the user; never rewrite the main branch to fix wording.

## Examples

```text
Bad title:   Update config
Good title:  Retry webhook delivery three times before marking it failed
```

```text
Bad body:
This PR fixes the bug we discussed. First I tried raising the timeout, but that did not help,
so I added retries. This is the fourth release this month, after billing-api 2.1.0.

Good body:
A webhook that timed out once was marked failed and never sent again. Delivery now retries
three times, after 1 s, 5 s and 30 s, before it is marked failed.

Breaking: `WEBHOOK_TIMEOUT_MS` now applies to each attempt, not to the whole delivery.

Checked: `bun test src/webhooks` passes. Not tested against a live endpoint.
```
