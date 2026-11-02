---
name: maintaining-project-records
description: "Keeps project memory in the repo: handoff, todo, questions, requirements documents and the rules for parallel agent sessions. Use when a repo has .agents/handoff.md, todo.md or questions.md and a session starts or resumes, a session ends, a sub-project closes out, the records are being set up or migrated, or several sessions run in parallel (lead, workers, claims, merging streams, shared-machine test locks)."
---

# Maintaining project records

Project memory lives in the repo, not in chat. One
home per kind of fact, updated in the same commit as the change it describes. Git history holds
everything else.

## Layout
```
AGENTS.md              thin entry point: read order, imports. No rules here.
.agents/AGENTS.md      conventions, hard rules, commands, review checklist
.agents/todo.md        open work only (plus deferred findings and later ideas)
.agents/questions.md   open user questions only
.agents/handoff.md     current state and workstreams, rewritten on every state change
docs/requirements/     one document per product area (or the repo's existing docs location)
docs/plans/            plans for active work only
```
- Root `AGENTS.md` says in a plain sentence where to start ("Open .agents/AGENTS.md first"); that
  is the portable path for any agent. The `@.agents/AGENTS.md` import syntax is Claude Code's:
  add it too when Claude Code is used.
- No `CLAUDE.md` beside the root `AGENTS.md`. Claude Code reads a root AGENTS.md only when no
  CLAUDE.md exists in the working directory or above, and never reads files under `.agents/` on its own.
- Reserve `.agents/memories/`, `skills/`, `agents/`, `tasks/`. Never create `.agents/agents.md`: it
  collides with `AGENTS.md` on case-insensitive file systems (Windows, default macOS).
- No ids anywhere (decision, work item, spec section). They point at files that get deleted.
- No worklog, history, learnings or mirror-of-the-code files. Skip speculative files few tools read (such as `llms.txt`).
- Requirements documents are complete enough to rebuild the feature from scratch. Each ends with a
  "Why it is this way" section: current behaviour and reasons, amended in place, no history,
  with accepted costs and rejected alternatives.

## Where each fact goes
| Event | Record |
|---|---|
| Behaviour decided or changed | the area's requirements document, reason in "Why it is this way" |
| Chose X over Y | same section: why X, why not Y |
| Deferred work or an unfixed finding | todo line only. A deferral is not a decision |
| Work finished | delete its todo line |
| Need user input | questions entry. Ask in the session that raised it |
| User answers | outcome into the owning document or a todo line, then delete the question |
| User states a durable rule | `.agents/AGENTS.md` and its review checklist |
| Same rule | owed tests to todo, update the affected plan in `docs/plans/`, sweep the code for violations |
| Trap that cost time | a rule in `.agents/AGENTS.md`, or a comment at the code |
| Stream assigned, committed, done, blocked | rewrite its handoff row now (sessions die) |

A question entry holds: what it is and what happens today, a recommendation with its reason,
alternatives with why and why not. No filler cons.

Before reporting a review finding, check todo and questions. Before planning from a tracked list
of problems, recompute it from the code; records drift.

Documents the user asks for that are not records (research, reviews) go in one ignored folder at
the repo root (for example `temp/`), never `docs/`. Tell the user the path. Durable lessons go in
`.agents/AGENTS.md` or the global instructions file, never only in chat or agent memory.

## Cold start
1. Read the root AGENTS.md read order, then `.agents/handoff.md`.
2. Reconcile with git: `git status`, `git log --oneline -5`, `git worktree list`, `git branch -vv`.
   Git wins: fix the handoff first and say what was stale.
3. Take the stream the user named. With none named you are the lead (see Parallel sessions).
   Never touch another stream's claimed files while it may be active: ask the user.

Test of the records: start a fresh agent with no chat history, ask what is in flight, what is next
and what is blocked, and compare with the handoff.

## End of session
1. Fold anything durable into its home (table above). Delete finished todo lines and answered questions.
2. Rewrite handoff.md: branch, workstreams (stream, owner, branch or worktree, claimed paths,
   state, next step), pending user calls, blockers. Never append.
3. Run the repo's line-width or format check on every edited record file.
4. Commit record changes with the code they describe.
5. Push after a sensible chunk of committed work. Force pushes, PRs and remote branch changes
   still need the user's go per action.

## Other situations
- Sub-project or long-running branch ends: open `references/closeout.md`.
- Setting up records in a new repo, or migrating off decision tables, worklog, history or
  learnings files: open `references/migrating-from-old-model.md`.
- Several sessions in one repo (one lead, named workers; the lead never edits code): open
  `references/parallel-sessions.md`. Before running tests or a gate on a shared machine, open
  `references/shared-machine-gates.md`.

## Easy to get wrong
- Appending to handoff instead of rewriting it, or updating it only at session end.
- Logging a deferral as a decision, or keeping a finished item in todo.
- Staging another stream's files from a shared checkout.
